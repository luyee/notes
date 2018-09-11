etcd put流程

client请求经过rpc通信之后调用Put()方法v3_server.go

```
//v3_server.go
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PutResponse), nil
}
//v3_server.go
func (s *EtcdServer) raftRequest(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	for {
		resp, err := s.raftRequestOnce(ctx, r)
		if err != auth.ErrAuthOldRevision {
			return resp, err
		}
	}
}
//v3_server.go
func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	result, err := s.processInternalRaftRequestOnce(ctx, r)
	if err != nil {
		return nil, err
	}
	if result.err != nil {
		return nil, result.err
	}
	return result.resp, nil
}
//v3_server.go
func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error) {
	ai := s.getAppliedIndex()
	ci := s.getCommittedIndex()
	if ci > ai+maxGapBetweenApplyAndCommitIndex {
		return nil, ErrTooManyRequests
	}

	r.Header = &pb.RequestHeader{
		ID: s.reqIDGen.Next(),
	}

	authInfo, err := s.AuthInfoFromCtx(ctx)
	if err != nil {
		return nil, err
	}
	if authInfo != nil {
		r.Header.Username = authInfo.Username
		r.Header.AuthRevision = authInfo.Revision
	}

	data, err := r.Marshal()
	if err != nil {
		return nil, err
	}

	if len(data) > int(s.Cfg.MaxRequestBytes) {
		return nil, ErrRequestTooLarge
	}

	id := r.ID
	if id == 0 {
		id = r.Header.ID
	}
	ch := s.w.Register(id)

	cctx, cancel := context.WithTimeout(ctx, s.Cfg.ReqTimeout())
	defer cancel()

	start := time.Now()
	err = s.r.Propose(cctx, data)
	if err != nil {
		proposalsFailed.Inc()
		s.w.Trigger(id, nil) // GC wait
		return nil, err
	}
	proposalsPending.Inc()
	defer proposalsPending.Dec()

	select {
	case x := <-ch:
		return x.(*applyResult), nil
	case <-cctx.Done():
		proposalsFailed.Inc()
		s.w.Trigger(id, nil) // GC wait
		return nil, s.parseProposeCtxErr(cctx.Err(), start)
	case <-s.done:
		return nil, ErrStopped
	}
}
```
这段主要看err = s.r.Propose(cctx, data)

```
//raft/node.go
func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}
//raft/node.go
func (n *node) stepWait(ctx context.Context, m pb.Message) error {
	return n.stepWithWaitOption(ctx, m, true)
}
//raft/node.go
func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {
	if m.Type != pb.MsgProp {
		select {
		case n.recvc <- m:
			return nil
		case <-ctx.Done():
			return ctx.Err()
		case <-n.done:
			return ErrStopped
		}
	}
	ch := n.propc
	pm := msgWithResult{m: m}
	if wait {
		pm.result = make(chan error, 1)
	}
	select {
	case ch <- pm:
		if !wait {
			return nil
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	select {
	case rsp := <-pm.result:
		if rsp != nil {
			return rsp
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	return nil
}
```

这里将request传到channel n.propc中，
```
ch := n.propc
pm := msgWithResult{m: m}
if wait {
  pm.result = make(chan error, 1)

}
select {
case ch <- pm:
  if !wait {
    return nil
  }
```

找到对应的propc就好了，这个在node(raft/node.go)的主循环run()中

```
//raft/node.go->run()
select {
// TODO: maybe buffer the config propose if there exists one (the way
// described in raft dissertation)
// Currently it is dropped in Step silently.
case pm := <-propc:
	m := pm.m
	m.From = r.id
	err := r.Step(m)
	if pm.result != nil {
		pm.result <- err
		close(pm.result)
	}
```
raft/raft.go

```
//func (r *raft) Step(m pb.Message) error {}<-->raft/raft.go
default:
	err := r.step(r, m)
	if err != nil {
		return err
	}
}
```
这个step依据不同的角色是不同的，分别对应stepCandidate，stepFollower，stepLeader，都是在bexx中赋值的

```
func (r *raft) becomeCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> candidate]")
	}
	r.step = stepCandidate
	r.reset(r.Term + 1)
	r.tick = r.tickElection
	r.Vote = r.id
	r.state = StateCandidate
	r.logger.Infof("%x became candidate at term %d", r.id, r.Term)
}
func (r *raft) becomeFollower(term uint64, lead uint64) {
	r.step = stepFollower
	r.reset(term)
	r.tick = r.tickElection
	r.lead = lead
	r.state = StateFollower
	r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}
func (r *raft) becomeLeader() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateFollower {
		panic("invalid transition [follower -> leader]")
	}
	r.step = stepLeader
	r.reset(r.Term)
	r.tick = r.tickHeartbeat
	r.lead = r.id
	r.state = StateLeader

	// Conservatively set the pendingConfIndex to the last index in the
	// log. There may or may not be a pending config change, but it's
	// safe to delay any future proposals until we commit all our
	// pending log entries, and scanning the entire tail of the log
	// could be expensive.
	r.pendingConfIndex = r.raftLog.lastIndex()

	r.appendEntry(pb.Entry{Data: nil})
	r.logger.Infof("%x became leader at term %d", r.id, r.Term)
}
func (r *raft) becomePreCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> pre-candidate]")
	}
	// Becoming a pre-candidate changes our step functions and state,
	// but doesn't change anything else. In particular it does not increase
	// r.Term or change r.Vote.
	r.step = stepCandidate
	r.votes = make(map[uint64]bool)
	r.tick = r.tickElection
	r.state = StatePreCandidate
	r.logger.Infof("%x became pre-candidate at term %d", r.id, r.Term)
}
```

