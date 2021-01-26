# Debugging a race in miniredis

I recently tracked down a race (which it turns out was already fixed upstream) in alicebob/miniredis.  Adam asked me how I tracked it down; this note describes how.  The short version is: since all I had were the goroutine stacktraces, I worked from there.  Only 2 were of interest; and it was easy to see what lock each was waiting on.  So from there, all I had to do was look at the library to figure out what could possibly be holding that lock; indeed in both cases it was the other goroutine.

## Orienting ourselves

From past attempts, I thought it unlikely that I'd be able to reproduce the issue locally, so I worked from what I had: the stacktraces from CI, reproduced [below](#panic-traces).  (First I got a little distracted figuring out that the lock-release-failed message right before was, if not unrelated, at least not a direct part of the deadlock.)  Let's go through the goroutines involved.

* 315 is the 10-minute test timeout itself.
* 1 is the main test-runner; it's just waiting on some test to complete (which is a chan receive, because tests run in goroutines).  94 is the same, but for a sub-test (via some testify stuff).
* 19, 297, 298, 302, and 303 seem to be unrelated background threads (in OpenCensus and gRPC); I don't know what they're doing but they don't seem super related and I think I've seen them before in other crash-dumps.  (The fact that we're pretty sure this has *something* to do with miniredis also suggests these are not as important.  But ultimately it was just a guess.)
* 284 and 305 both mention miniredis, so I looked more closely at those.

First, I just dug through what each thread was doing; neither is very complicated.  (Indeed the saving grace of this whole bug is that there's only so much code involved.)  Goroutine 284 the top 5 frames are the interesting ones: we call `Miniredis.Close()` which calls `Server.Close()` which is waiting on `WaitGroup.Wait()`.  Meanwhile, goroutine 305 seems to be in the redis server's accept-loop: it has accepted a connection, which is handling a command -- in our case DEL.  It seems the first thing each command does is check auth if necessary (it's not, for us); and it's blocking on the lock at the start of that check.

## Where's the deadlock?

Now obviously we have a deadlock, so something needs to be holding each of those locks.  It could either be one of the other goroutines, or some goroutine that panicked or otherwise exited without unlocking the lock.  Let's look at the waitgroup first; it must be waiting on some who has called `wg.Add()` but not (yet) `wg.Done()`.  There are only 3 callers in the whole module; we can quickly see the one in `redis.go` is a totally different waitgroup (actually honestly I only grepped in the package so I missed this one) and the one in `NewServer` doesn't fit any of the goroutines that are running (and since it does a `defer` it must not have panicked).  But the one in `ServeConn` exactly matches where goroutine 305 is.  That makes some sense: close is waiting for all the connection-handlers to finish.

So now we need to know what goroutine 305 -- the lock-acquire in `handleAuth` -- is waiting on.  This is where I got stuck for a little bit.  It wasn't obvious who would be holding it.  (Actually, it is, but we'll get to that later.)  I got confused for a bit finding my bearings in the code; there are two `Close()` methods at issue (turns out I spent most of my time looking at the wrong one), and both `Server` and `Miniredis` have a lock, and `RedisDB` has a pointer to the latter.  (Another red herring was whether some other code in our codebase was using `ServerForTests` to do something bad; in principle it exposes the lock!)  But eventually, I looked through all the callers of `Lock()` in the package, and it didn't seem to me that any of them could have exited leaving the lock locked (most did `m.Lock(); defer m.Unlock()` which is a very easy-to-see guarantee of that).  So it had to be one of the still-running goroutines.

And there's only one other culprit -- 284, which is in close.  So I looked back, more carefully this time, at each of its frames within miniredis; luckily there are only two, `Miniredis.Close` and `Server.Close`!  And indeed, with fresh eyes, `Miniredis.Close` holds exactly the lock we are locking for.  So that's how the DEL handler is waiting on Close.

## Putting it all together

So now we can piece back together what happened: the connection-handler took out its slot in the waitgroup, then `Close()` took out `Miniredis.Lock`, and then each waited for the other.  Indeed this matches what we had vaguely assumed: it was due to sending a command around the same time as the `Close()` request.

## Panic traces

```
goroutine 315 [running]:
testing.(*M).startAlarm.func1()
	/usr/lib/go-1.14/src/testing/testing.go:1509 +0x11c
created by time.goFunc
	/usr/lib/go-1.14/src/time/sleep.go:168 +0x52

goroutine 1 [chan receive, 9 minutes]:
testing.(*T).Run(0xc0004f0360, 0x197eede, 0xa, 0x19c3b08, 0x1)
	/usr/lib/go-1.14/src/testing/testing.go:1091 +0x739
testing.runTests.func1(0xc0004f0360)
	/usr/lib/go-1.14/src/testing/testing.go:1334 +0xa7
testing.tRunner(0xc0004f0360, 0xc0005bfd38)
	/usr/lib/go-1.14/src/testing/testing.go:1039 +0x1ec
testing.runTests(0xc000524060, 0x2819ac0, 0x5, 0x5, 0x0)
	/usr/lib/go-1.14/src/testing/testing.go:1332 +0x528
testing.(*M).Run(0xc000096000, 0x0)
	/usr/lib/go-1.14/src/testing/testing.go:1249 +0x440
main.main()
	_testmain.go:52 +0x224

goroutine 19 [select]:
go.opencensus.io/stats/view.(*worker).start(0xc00018e500)
	/home/ubuntu/go/pkg/mod/go.opencensus.io@v0.22.5/stats/view/worker.go:276 +0x1d6
created by go.opencensus.io/stats/view.init.0
	/home/ubuntu/go/pkg/mod/go.opencensus.io@v0.22.5/stats/view/worker.go:34 +0xaf

goroutine 298 [chan receive, 9 minutes]:
google.golang.org/grpc.(*addrConn).resetTransport(0xc000a2eb00)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/clientconn.go:1213 +0x88b
created by google.golang.org/grpc.(*addrConn).connect
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/clientconn.go:843 +0x104

goroutine 94 [chan receive, 9 minutes]:
testing.(*T).Run(0xc000a41560, 0x16a0a34, 0x12, 0xc0004d9b90, 0xc000a41501)
	/usr/lib/go-1.14/src/testing/testing.go:1091 +0x739
github.com/stretchr/testify/suite.runTests(0x1dbfdc0, 0xc000a41560, 0xc000965a40, 0x8, 0x8)
	/home/ubuntu/go/pkg/mod/github.com/stretchr/testify@v1.6.1/suite/suite.go:203 +0xe7
github.com/stretchr/testify/suite.Run(0xc000a41560, 0x1d81ae0, 0xc0007ccf50)
	/home/ubuntu/go/pkg/mod/github.com/stretchr/testify@v1.6.1/suite/suite.go:176 +0x8c4
github.com/Khan/webapp/dev/servicetest.Run(...)
	/home/ubuntu/webapp-workspace/webapp/dev/servicetest/suite.go:218
github.com/Khan/webapp/services/users/creation.TestCreate(0xc000a41560)
	/home/ubuntu/webapp-workspace/webapp/services/users/creation/create_test.go:217 +0x5f
testing.tRunner(0xc000a41560, 0x19c3b08)
	/usr/lib/go-1.14/src/testing/testing.go:1039 +0x1ec
created by testing.(*T).Run
	/usr/lib/go-1.14/src/testing/testing.go:1090 +0x701

goroutine 303 [select, 9 minutes]:
google.golang.org/grpc/internal/transport.(*controlBuffer).get(0xc000c8b9a0, 0x1, 0x0, 0x0, 0x0, 0x0)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/controlbuf.go:395 +0x1fe
google.golang.org/grpc/internal/transport.(*loopyWriter).run(0xc000babbc0, 0x0, 0x0)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/controlbuf.go:515 +0x2c2
google.golang.org/grpc/internal/transport.newHTTP2Client.func3(0xc0004be540)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/http2_client.go:376 +0x12b
created by google.golang.org/grpc/internal/transport.newHTTP2Client
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/http2_client.go:374 +0x1892

goroutine 302 [IO wait, 9 minutes]:
internal/poll.runtime_pollWait(0x7fa2309cd030, 0x72, 0x1d79620)
	/usr/lib/go-1.14/src/runtime/netpoll.go:203 +0x55
internal/poll.(*pollDesc).wait(0xc000c94018, 0x72, 0x8000, 0x8000, 0xffffffffffffffff)
	/usr/lib/go-1.14/src/internal/poll/fd_poll_runtime.go:87 +0xe4
internal/poll.(*pollDesc).waitRead(...)
	/usr/lib/go-1.14/src/internal/poll/fd_poll_runtime.go:92
internal/poll.(*FD).Read(0xc000c94000, 0xc000a6a000, 0x8000, 0x8000, 0x0, 0x0, 0x0)
	/usr/lib/go-1.14/src/internal/poll/fd_unix.go:169 +0x253
net.(*netFD).Read(0xc000c94000, 0xc000a6a000, 0x8000, 0x8000, 0x11, 0x0, 0x0)
	/usr/lib/go-1.14/src/net/fd_unix.go:202 +0x66
net.(*conn).Read(0xc0007ba740, 0xc000a6a000, 0x8000, 0x8000, 0xc000a6a000, 0x9, 0x4813e0)
	/usr/lib/go-1.14/src/net/net.go:184 +0xa1
bufio.(*Reader).Read(0xc000babaa0, 0xc0000b7d18, 0x9, 0x9, 0x800010601, 0x1060100000000, 0x49dd82)
	/usr/lib/go-1.14/src/bufio/bufio.go:226 +0x827
io.ReadAtLeast(0x1d712c0, 0xc000babaa0, 0xc0000b7d18, 0x9, 0x9, 0x9, 0x0, 0xc000c8ba08, 0xc000075cb0)
	/usr/lib/go-1.14/src/io/io.go:310 +0x99
io.ReadFull(...)
	/usr/lib/go-1.14/src/io/io.go:329
golang.org/x/net/http2.readFrameHeader(0xc0000b7d18, 0x9, 0x9, 0x1d712c0, 0xc000babaa0, 0x0, 0x0, 0xc000c8b9ef, 0xc000c8b9a0)
	/home/ubuntu/go/pkg/mod/golang.org/x/net@v0.0.0-20201209123823-ac852fbbde11/http2/frame.go:237 +0xac
golang.org/x/net/http2.(*Framer).ReadFrame(0xc0000b7ce0, 0xc000dc2680, 0xc000dc2680, 0x0, 0x0)
	/home/ubuntu/go/pkg/mod/golang.org/x/net@v0.0.0-20201209123823-ac852fbbde11/http2/frame.go:492 +0xfc
google.golang.org/grpc/internal/transport.(*http2Client).reader(0xc0004be540)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/http2_client.go:1317 +0x282
created by google.golang.org/grpc/internal/transport.newHTTP2Client
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/internal/transport/http2_client.go:330 +0x13f3

goroutine 284 [semacquire, 9 minutes]:
sync.runtime_Semacquire(0xc000cc7110)
	/usr/lib/go-1.14/src/runtime/sema.go:56 +0x42
sync.(*WaitGroup).Wait(0xc000cc7108)
	/usr/lib/go-1.14/src/sync/waitgroup.go:130 +0xd4
github.com/alicebob/miniredis/server.(*Server).Close(0xc000cc70e0)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/server/server.go:109 +0x164
github.com/alicebob/miniredis.(*Miniredis).Close(0xc000c60cc0)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/miniredis.go:161 +0xb6
github.com/Khan/webapp/dev/servicetest.(*Suite)._initMemorystoreClient.func1()
	/home/ubuntu/webapp-workspace/webapp/dev/servicetest/suite.go:95 +0x71
github.com/Khan/webapp/dev/khantest.(*Suite).TearDownTest(0xc0007ccf50)
	/home/ubuntu/webapp-workspace/webapp/dev/khantest/suite.go:70 +0xcb
github.com/stretchr/testify/suite.Run.func1.1(0xc00097d988, 0xc000c8fb00, 0x16a0a34, 0x12, 0x0, 0x0, 0x1dc9fe0, 0xc000baa480, 0xc000baa480, 0xc00011fcd0, ...)
	/home/ubuntu/go/pkg/mod/github.com/stretchr/testify@v1.6.1/suite/suite.go:141 +0x123
github.com/stretchr/testify/suite.Run.func1(0xc000c8fb00)
	/home/ubuntu/go/pkg/mod/github.com/stretchr/testify@v1.6.1/suite/suite.go:159 +0x393
testing.tRunner(0xc000c8fb00, 0xc0004d9b90)
	/usr/lib/go-1.14/src/testing/testing.go:1039 +0x1ec
created by testing.(*T).Run
	/usr/lib/go-1.14/src/testing/testing.go:1090 +0x701

goroutine 297 [select, 9 minutes]:
google.golang.org/grpc.(*ccBalancerWrapper).watcher(0xc00062e100)
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/balancer_conn_wrappers.go:69 +0x19c
created by google.golang.org/grpc.newCCBalancerWrapper
	/home/ubuntu/go/pkg/mod/google.golang.org/grpc@v1.34.0/balancer_conn_wrappers.go:60 +0x2f4

goroutine 305 [semacquire, 9 minutes]:
sync.runtime_SemacquireMutex(0xc000c60cc4, 0x900000000, 0x1)
	/usr/lib/go-1.14/src/runtime/sema.go:71 +0x47
sync.(*Mutex).lockSlow(0xc000c60cc0)
	/usr/lib/go-1.14/src/sync/mutex.go:138 +0x1c1
sync.(*Mutex).Lock(0xc000c60cc0)
	/usr/lib/go-1.14/src/sync/mutex.go:81 +0x7d
github.com/alicebob/miniredis.(*Miniredis).handleAuth(0xc000c60cc0, 0xc000d5b2e0, 0x7fa1eb650000)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/miniredis.go:314 +0x5b
github.com/alicebob/miniredis.(*Miniredis).cmdDel(0xc000c60cc0, 0xc000d5b2e0, 0xc000cf9bb5, 0x3, 0xc000d5bd70, 0x1, 0x1)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/cmd_generic.go:191 +0x47
github.com/alicebob/miniredis/server.(*Server).dispatch(0xc000cc70e0, 0xc000d5b2e0, 0xc000d5bd70, 0x1, 0x2)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/server/server.go:158 +0x259
github.com/alicebob/miniredis/server.(*Server).servePeer(0xc000cc70e0, 0x1db2540, 0xc0007bac48)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/server/server.go:135 +0x1ee
github.com/alicebob/miniredis/server.(*Server).ServeConn.func1(0xc000cc70e0, 0x1db2540, 0xc0007bac48)
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/server/server.go:79 +0x1e2
created by github.com/alicebob/miniredis/server.(*Server).ServeConn
	/home/ubuntu/go/pkg/mod/github.com/alicebob/miniredis@v2.5.0+incompatible/server/server.go:71 +0x7e
```
