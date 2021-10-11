1. 配置好拦截器需要拦截哪些请求
2. 配置放在容器中

重写 preHandler，



原理

1. 根据当前请求，找到能处理请求的Handler，找到该handler下的所有拦截器
2. 先来顺序执行所有拦截器的prehandle
   1. 如果prehanlder返回为true，则执行下一个拦截器的prehandler
   2. 如果拦截器返回false，则倒叙执行aftercompetition
3. 如果任何一个拦截器返回false，则会跳出，不执行目标方法
4. 如果所有拦截器返回 true，则会执行目标方法
5. 倒叙执行所有拦截器的posthandler
6. 前面都步骤有任何异常都会触发aftercompetition
7. 得到返回值后都会倒叙出发aftercompetition