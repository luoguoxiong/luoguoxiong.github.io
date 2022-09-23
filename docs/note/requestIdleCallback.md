# requestIdleCallback 实现

```js
const requestIdleCallback = (cb) => {
  const start = Date.now();
  return setTimeout(function() {
    cb({
      timeRemaining: function() {
        return Math.max(0, 60 - (Date.now() - start));
      },
    });
  }, 0);
};

const cancelIdleCallback = function(id) {
  return clearTimeout(id);
};

const runTask = (taskName) => {
  let i = 0;
  while(i < 1000000000){
    i++;
  }
};

const tasks = [runTask, runTask, runTask];
function myNonEssentialWork(deadline) {
  console.log(`第${tasks.length}个任务，${deadline.timeRemaining() > 0 ? '' : '不'}可以跑了`);
  if(deadline.timeRemaining() > 0){
    if(tasks.length > 0){
      tasks.length > 1 && requestIdleCallback(myNonEssentialWork);
      const task = tasks.pop();
      task(tasks.length);
    }
  }else{
    requestIdleCallback(myNonEssentialWork);
  }
}
requestIdleCallback(myNonEssentialWork);
// 第3个任务，可以跑了
// 第2个任务，不可以跑了
// 第2个任务，可以跑了
// 第1个任务，不可以跑了
// 第1个任务，可以跑了
```