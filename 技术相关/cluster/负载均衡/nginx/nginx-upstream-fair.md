# nginx的第三方负载均衡模块upstream_fair_module 公平负载均衡策略相关

因为目前的web集群中存在一种比较严重的问题，即
单个节点卡，整个web集群卡。
当然这里有一个前提是整个web 集群采用nginx进行负载均衡，且用默认的负载均衡策略（RR轮询策略）
最简单的还原办法是：
代码启动两个节点，在一个节点中写一个请求拦截器让每个请求休眠1s
这时候就可以明显的感觉到整个web集群的卡顿。
还有一种单个节点因为某个计算请求，使一个节点卡顿，半瘫痪，整个web集群因为采用
RR负载均衡策略而导致卡顿。更严重的是不可用
在上面说的情况下，web集群并没有达到随着节点数的扩展，整个性能得到扩展，或者只有节点可用整个集群可用的目的。
虽然nginx负载均衡的代码并不在于web集群主题代码范围之内，但是保障web集群功能的正常交付本身就是web集群应该做的事情，所以说至少能寻找一种代替RR策略的nginx负载均衡策略或者是别的负载均衡器。

网上主推的一种策略是fair策略但是，编译，调试之后并没有达到目前想要达到的目的，所以才分析了一番。
结论：
这里的公平策略并不是web集群需要的公平策略。
下面的分析会省略一些次要的代码
## 整体流程
整个负载均衡策略重要的三部分分别是
1: 负载均衡策略的初始化，可以先简单的看成是初始化整个fair策略所需要的资源，这里的资源包括告诉nginx要调用什么函数选择节点，用什么函数处理节点请求之后事情，内存空间的申请等。
2: 选择一个节点交由nginx进行处理
3: 节点处理请求之后的后续处理。nginx获取到一个节点，然后进行请求的转发之后，再交由fair策略进行一些资源释放等操作。
上面的三部分分别由下面三个函数提供。
1：ngx_http_upstream_init_fair_peer
2：ngx_http_upstream_get_fair_peer
3：ngx_http_upstream_free_fair_peer
## 选择节点
先看
```
ngx_int_t
ngx_http_upstream_get_fair_peer(ngx_peer_connection_t *pc, void *data)
{
    ngx_int_t                           ret;
    ngx_uint_t                          peer_id, i;
    ngx_http_upstream_fair_peer_data_t *fp = data;
    ngx_http_upstream_fair_peer_t      *peer;
    ngx_atomic_t                       *lock;
    ...
    ret = ngx_http_upstream_choose_fair_peer(pc, fp, &peer_id);
    ...
    pc->sockaddr = peer->sockaddr;
    pc->socklen = peer->socklen;
    pc->name = &peer->name;
    ...
    peer->shared->last_req_id = fp->peers->shared->total_requests;
    ngx_http_upstream_fair_update_nreq(fp, 1, pc->log);
    peer->shared->total_req++;
    return ret;
}
```
从ngx_http_upstream_choose_fair_peer方法中获取一个节点。上面的ret可以看成是一个选中节点的信息，然后赋值给pc，这里的pc可以认为这是告诉nginx要连接哪个节点。
然后就是更新节点的一些全局信息如选中的节点正在处理的请求数（nreq）加1，总请求数（total_req）加1。
继续看ngx_http_upstream_choose_fair_peer

```
static ngx_int_t
ngx_http_upstream_choose_fair_peer(ngx_peer_connection_t *pc,
    ngx_http_upstream_fair_peer_data_t *fp, ngx_uint_t *peer_id)
{
    ngx_uint_t                          npeers;
    ngx_uint_t                          best_idx = NGX_PEER_INVALID;
    ngx_uint_t                          weight_mode;

    npeers = fp->peers->number;
    weight_mode = fp->peers->weight_mode;

    if (npeers == 1) {
        *peer_id = 0;
        return NGX_OK;
    }

    best_idx = ngx_http_upstream_choose_fair_peer_idle(pc, fp);
    if (best_idx != NGX_PEER_INVALID) {
        goto chosen;
    }
    best_idx = ngx_http_upstream_choose_fair_peer_busy(pc, fp);
    if (best_idx != NGX_PEER_INVALID) {
        goto chosen;
    }

    return NGX_BUSY;

chosen:
    ngx_bitvector_set(fp->tried, best_idx);
    if (weight_mode == WM_DEFAULT) {
        ngx_http_upstream_fair_peer_t      *peer = &fp->peers->peer[best_idx];

        if (peer->shared->current_weight-- == 0) {
            peer->shared->current_weight = peer->weight;
        }
    }
    return NGX_OK;
}
```
1:如果是一个节点则直接返回。
2:先从从空闲的节点中选取ngx_http_upstream_choose_fair_peer_idle
3:再从繁忙的节点中选取ngx_http_upstream_choose_fair_peer_busy
4:如果成功选择一个则走到chosen:
    4.1 设置ngx_bitvector_set 可以简单认为是标识这个节点已经被选择了。
    4.2 如果是WM_DEFAULT模式（fair负载均衡模式可以选择几种策略，可以先忽略）则权重更新。
5:先忽略一些异常的情况，如返回NGX_BUSY nginx 会进行怎样的处理。先主关注1和2

接着看ngx_http_upstream_choose_fair_peer_idle 从空闲的节点中挑选一个节点。

```
static ngx_uint_t
ngx_http_upstream_choose_fair_peer_idle(ngx_peer_connection_t *pc,
    ngx_http_upstream_fair_peer_data_t *fp)
{
    ngx_uint_t                          i, n;
    ngx_uint_t                          npeers = fp->peers->number;
    ngx_uint_t                          weight_mode = fp->peers->weight_mode;
    ngx_uint_t                          best_idx = NGX_PEER_INVALID;
    ngx_uint_t                          best_nreq = ~0U;

    for (i = 0, n = fp->current; i < npeers; i++, n = (n + 1) % npeers) {
        ngx_uint_t nreq = fp->peers->peer[n].shared->nreq;
        ngx_uint_t weight = fp->peers->peer[n].weight;

        if (fp->peers->peer[n].shared->fails > 0)
            continue;

        if (nreq >= weight || (nreq > 0 && weight_mode != WM_IDLE)) {
            continue;
        }

        if (ngx_http_upstream_fair_try_peer(pc, fp, n) != NGX_OK) {
            continue;
        }

        /* not in WM_IDLE+no_rr mode: the first completely idle backend gets chosen */
        if (weight_mode != WM_IDLE || !fp->peers->no_rr) {
            best_idx = n;
            break;
        }

        /* in WM_IDLE+no_rr mode we actually prefer slightly loaded backends
         * to totally idle ones, under the assumption that they're spawned
         * on demand and can handle up to 'weight' concurrent requests
         */
        if (best_idx == NGX_PEER_INVALID || nreq) {
            if (best_nreq <= nreq) {
                continue;
            }
            best_idx = n;
            best_nreq = nreq;
        }
    }

    return best_idx;
}
```
1:遍寻每个节点
2:如果节点已经失败（这里的失败可以见下面节点处理请求之后处理分析。）则跳过
3:如果当前节点正在处理的请求数大于权重，或者fair的模式是WM_IDLE（空闲模式）且当前节点已经有处理的请求了则跳过
要理解3需要知道fair可以进行下面的配置
```
upstream backend {
    server backend1.example1.com weight=4;
    server backend2.example1.com weight=3;
    server backend3.example1.com weight=4;
    fair weight_mode=idle no_rr;
}
```
其中还有weight_mode=peak等模式
4：尝试从ngx_http_upstream_fair_try_peer里面获取
5：尝试成功之后，对于不是WM_IDLE且关闭轮询的情况下，会返回这个尝试成功的服务器。
6：在WM_IDLE和no_rr开启的情况下，会根据当前所承载的请求数来轮询选优，选择承载请求数最少的服务器。（注意，这里方法里面best_idx = NGX_PEER_INVALID默认设置。则
```
if (best_idx == NGX_PEER_INVALID || nreq) {
            if (best_nreq <= nreq) {
                continue;
            }
            best_idx = n;
            best_nreq = nreq;
        }
```
这一段等价于获取当前正在处理的连接最少的节点。）
7：还是直接看ngx_http_upstream_fair_try_peer
```
static ngx_int_t
ngx_http_upstream_fair_try_peer(ngx_peer_connection_t *pc,
    ngx_http_upstream_fair_peer_data_t *fp,
    ngx_uint_t peer_id)
{
    ngx_http_upstream_fair_peer_t        *peer;

    if (ngx_bitvector_test(fp->tried, peer_id))
        return NGX_BUSY;

    peer = &fp->peers->peer[peer_id];

    if (!peer->down) {

#if (NGX_HTTP_UPSTREAM_CHECK)
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "[upstream_fair] get fair peer, check_index: %ui",
                       peer->check_index);

        if (!ngx_http_upstream_check_peer_down(peer->check_index)) {
#endif

        if (peer->max_fails == 0 || peer->shared->fails < peer->max_fails) {
            return NGX_OK;
        }

        if (ngx_time() - peer->accessed > peer->fail_timeout) {
            ngx_log_debug3(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] resetting fail count for peer %d, time delta %d > %d",
                peer_id, ngx_time() - peer->accessed, peer->fail_timeout);
            peer->shared->fails = 0;
            return NGX_OK;
        }

#if (NGX_HTTP_UPSTREAM_CHECK)
        }
#endif

    }

    return NGX_BUSY;
}
```
1:如果节点已经被使用（见上面ngx_http_upstream_choose_fair_peer chosen分析）直接放弃该节点
2：看到这里有一个#if (NGX_HTTP_UPSTREAM_CHECK)先忽略这个，这个是说如果有健康检查模块先检查一下当前节点是否健康的。这是另一个主题了，先进行忽略。
然后看到最外层的!peer->down 判断，说的是判断当前节点是否能使用。
3:如果允许节点失败的次数为0（意思为不限制）或者当前失败的次数小于允许失败的次数则先假装这个节点合格，因为在上层还会根据当前节点的正在处理的连接数进行判断。
4：如果当前的时间减去节点时间的差值大于配置的最大超时时间，则允许再试一次。可以见nginx.conf配置`server 127.0.0.1:8081 max_fails=5 fail_timeout=200s;`

整个**ngx_http_upstream_fair_try_peer**方法分析完毕可以认为是
选择失败次数小于最大尝试次数的或者虽然超过最大尝试次数但是时间超过了最大超时时间的。这在注释中写的是**the core of load balancing logic** .....
也就是说所谓的fair就是在这里面挑选出最好的.....。

接着看
```
static ngx_int_t
ngx_http_upstream_choose_fair_peer_busy(ngx_peer_connection_t *pc,
    ngx_http_upstream_fair_peer_data_t *fp)
{
    ngx_uint_t                          i, n;
    ngx_uint_t                          npeers = fp->peers->number;
    ngx_uint_t                          weight_mode = fp->peers->weight_mode;
    ngx_uint_t                          best_idx = NGX_PEER_INVALID;
    ngx_uint_t                          sched_score;
    ngx_uint_t                          best_sched_score = ~0UL;

    /*
     * calculate sched scores for all the peers, choosing the lowest one
     */
    for (i = 0, n = fp->current; i < npeers; i++, n = (n + 1) % npeers) {
        ngx_http_upstream_fair_peer_t      *peer;
        ngx_uint_t                          nreq;
        ngx_uint_t                          weight;

        peer = &fp->peers->peer[n];
        nreq = fp->peers->peer[n].shared->nreq;

        if (weight_mode == WM_PEAK && nreq >= peer->weight) {
            ngx_log_debug3(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] backend %d has nreq %ui >= weight %ui in WM_PEAK mode", n, nreq, peer->weight);
            continue;
        }

        if (ngx_http_upstream_fair_try_peer(pc, fp, n) != NGX_OK) {
            if (!pc->tries) {
                ngx_log_debug(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] all backends exhausted");
                return NGX_PEER_INVALID;
            }

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] backend %d already tried", n);
            continue;
        }

        sched_score = ngx_http_upstream_fair_sched_score(pc, fp, n);

        if (weight_mode == WM_DEFAULT) {
            /*
             * take peer weight into account
             */
            weight = peer->shared->current_weight;
            if (peer->max_fails) {
                ngx_uint_t mf = peer->max_fails;
                weight = peer->shared->current_weight * (mf - peer->shared->fails) / mf;
            }
            if (weight > 0) {
                sched_score /= weight;
            }
            ngx_log_debug8(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] bss = %ui, ss = %ui (n = %d, w = %d/%d, f = %d/%d, weight = %d)",
                best_sched_score, sched_score, n, peer->shared->current_weight, peer->weight, peer->shared->fails, peer->max_fails, weight);
        }

        if (sched_score <= best_sched_score) {
            best_idx = n;
            best_sched_score = sched_score;
        }
    }

    return best_idx;
}
```
和上面的ngx_http_upstream_choose_fair_peer_idle一样的套路
几个不同点
1：排除的是WM_PEAK模式下节点正在处理的请求大于权重的节点
2：选择最优的不再是正在处理的连接数最少的而是ngx_http_upstream_fair_sched_score 评分函数计算出来的值最优的评分节点。且如果是WM_DEFAULT模式，则评分还需要再除以自身的权重。
直接看
```
#define SCHED_COUNTER_BITS 20
#define SCHED_NREQ_MAX ((~0UL) >> SCHED_COUNTER_BITS)
#define SCHED_COUNTER_MAX ((1 << SCHED_COUNTER_BITS) - 1)
#define SCHED_SCORE(nreq,delta) (((nreq) << SCHED_COUNTER_BITS) | (~(delta) & SCHED_COUNTER_MAX))
#define ngx_upstream_fair_min(a,b) (((a) < (b)) ? (a) : (b))

static ngx_uint_t
ngx_http_upstream_fair_sched_score(ngx_peer_connection_t *pc,
    ngx_http_upstream_fair_peer_data_t *fp,
    ngx_uint_t n)
{
    ngx_http_upstream_fair_peer_t      *peer = &fp->peers->peer[n];
    ngx_http_upstream_fair_shared_t    *fs = peer->shared;
    ngx_uint_t req_delta = fp->peers->shared->total_requests - fs->last_req_id;

    /* sanity check */
    if ((ngx_int_t)fs->nreq < 0) {
        ngx_log_error(NGX_LOG_WARN, pc->log, 0, "[upstream_fair] upstream %ui has negative nreq (%i)", n, fs->nreq);
        return SCHED_SCORE(0, req_delta);
    }

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, pc->log, 0, "[upstream_fair] peer %ui: nreq = %i, req_delta = %ui", n, fs->nreq, req_delta);

    return SCHED_SCORE(
        ngx_upstream_fair_min(fs->nreq, SCHED_NREQ_MAX),
        ngx_upstream_fair_min(req_delta, SCHED_COUNTER_MAX));
}
```
几点需要注意
1:    ngx_uint_t req_delta = fp->peers->shared->total_requests - fs->last_req_id;
（total_requests）整个nginx处理的请求数
（last_req_id）当前节点处理的最后一次处理的请求是整个nginx处理的所有请求的第几个。在一个节点选中交给nginx之前，fair 会把last_req_id置为total_requests，两者相减以为着节点上一次处理的请求到现在，整个nginx还处理了多少请求。
2:最终的分数为节点正在处理的请求数和节点上一次处理的请求到现在，整个nginx还处理了多少请求数的函数。两个值越小分值越高。


上面两个选择方式可以总结为
1、IDLE选择：        
IDLE选择的时候，对于有失败过的服务器，或者当前服务器所承载的请求数过多（大于权重，不是WM_IDLE模式时候有请求就算），就放弃选择该服务器。      
当该服务器所承载的请求在可接受范围内，就会接着进行尝试，尝试部分和busy时候的尝试是相同的。        
尝试成功之后，对于不是WM_IDLE且关闭轮询的情况下，会返回这个尝试成功的服务器。        
在WM_IDLE和no_rr开启的情况下，会根据当前所承载的请求数来轮询选优，选择承载请求数最少的服务器。
2、BUSY选择：        
在BUSY选择过程中，如果是WM_PEAK模式，会保证当前服务器承载的请求数小于权重的值。        
接着对候选服务器进行尝试，尝试成功的话就去计算候选服务器的评分，并选择分数最低的服务器。（评分规则建上面分析。）        
如果是WM_DEFAULT模式，计算出来的评分还会进行加权，这里权重的计算会根据失败次数而变化。最后实际分数会除以权值。








## 节点处理请求之后处理。
```
void
ngx_http_upstream_free_fair_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_fair_peer_data_t     *fp = data;
    ngx_http_upstream_fair_peer_t          *peer;
    ngx_atomic_t                           *lock;
    
    if (fp->current == NGX_PEER_INVALID) {
        return;
    }
    lock = &fp->peers->shared->lock;
    ngx_spinlock(lock, ngx_pid, 1024);
    if (!ngx_bitvector_test(fp->done, fp->current)) {
        ngx_bitvector_set(fp->done, fp->current);
        ngx_http_upstream_fair_update_nreq(fp, -1, pc->log);
    }
    if (fp->peers->number == 1) {
        pc->tries = 0;
    }
    if (state & NGX_PEER_FAILED) {
        peer = &fp->peers->peer[fp->current];
        peer->shared->fails++;
        peer->accessed = ngx_time();
    }
    ngx_spinlock_unlock(lock);
}
```
主要关注下面几点
1：ngx_http_upstream_fair_update_nreq(fp, -1, pc->log) 节点正在处理的请求数减一
2：如果选中的节点的处理返回状态为失败则失败次数加1
3：NGX_PEER_FAILED的状态可以是自己定义的50x系列请求等。
