第三周 并行编程 作业

基于 errgroup 实现一个 http server 的启动和关闭 ，以及 linux signal 信号的注册和处理，要保证能够一个退出，全部注销退出。

参考 [Go并发编程(七) 深入理解 errgroup](https://mp.weixin.qq.com/s?__biz=MzA3MzQ0MzI3MA==&mid=2454091882&idx=2&sn=aee5811bc29297fa83141b728892ca22&chksm=88bc2ac3bfcba3d54390d550180088d27efe3608dfb5711d5735656e906bb6be2c36632f2eff&scene=132#wechat_redirect)

```
func main() {
 g, ctx := errgroup.WithContext(context.Background())

 mux := http.NewServeMux()
 mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
  w.Write([]byte("pong"))
 })

 // 模拟单个服务错误退出
 serverOut := make(chan struct{})
 mux.HandleFunc("/shutdown", func(w http.ResponseWriter, r *http.Request) {
  serverOut <- struct{}{}
 })

 server := http.Server{
  Handler: mux,
  Addr:    ":8080",
 }

 // g1
 // g1 退出了所有的协程都能退出么？
 // g1 退出后, context 将不再阻塞，g2, g3 都会随之退出
 // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出
 g.Go(func() error {
  return server.ListenAndServe()
 })

 // g2
 // g2 退出了所有的协程都能退出么？
 // g2 退出时，调用了 shutdown，g1 会退出
 // g2 退出后, context 将不再阻塞，g3 会随之退出
 // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出
 g.Go(func() error {
  select {
  case <-ctx.Done():
   log.Println("errgroup exit...")
  case <-serverOut:
   log.Println("server will out...")
  }

  timeoutCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
  // 这里不是必须的，但是如果使用 _ 的话静态扫描工具会报错，加上也无伤大雅
  defer cancel()

  log.Println("shutting down server...")
  return server.Shutdown(timeoutCtx)
 })

 // g3
 // g3 捕获到 os 退出信号将会退出
 // g3 退出了所有的协程都能退出么？
 // g3 退出后, context 将不再阻塞，g2 会随之退出
 // g2 退出时，调用了 shutdown，g1 会退出
 // 然后 main 函数中的 g.Wait() 退出，所有协程都会退出
 g.Go(func() error {
  quit := make(chan os.Signal, 0)
  signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

  select {
  case <-ctx.Done():
   return ctx.Err()
  case sig := <-quit:
   return errors.Errorf("get os signal: %v", sig)
  }
 })

 fmt.Printf("errgroup exiting: %+v\n", g.Wait())
}
```
