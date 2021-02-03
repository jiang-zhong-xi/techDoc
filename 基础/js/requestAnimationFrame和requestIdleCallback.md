### requestAnimationFrame

这个API告诉浏览器希望执行一个回调函数，并且希望在本次js代码执行后、下次paint前执行回调函数。

### requestIdleCallback

这个API告诉浏览器希望浏览器在浏览器每一帧的空闲时间执行回调，而js代码执行然后paint期间的时间都不算入空闲时间。可以有一个参数，表明这段时间内如果没有执行那么在这一帧强制执行虽然这可能会导致失帧。

### Vs

![image-20210129170354809](C:\Users\walker\AppData\Roaming\Typora\typora-user-images\image-20210129170354809.png)

![image-20210129170450645](C:\Users\walker\AppData\Roaming\Typora\typora-user-images\image-20210129170450645.png)

可以看到requestAnimationFrame和requestIdleCallback后，requestAnimationFrame的回调在js脚本执行完一段时候开始执行，然后开始绘制、合成图层，接着是requestIdleCallback的回调函数开始执行。

Task Vs MicroTask Vs rAF Vs rLC

microTask在js同步代码执行后立马执行，Task则在本次绘制完成后执行，rAF在js代码执行后、本次绘制完成前执行，rLC则在本帧有空闲时间时才执行。所以对于DOM操作推荐在MicroTask或者rAF里执行，因为此时还没paint。