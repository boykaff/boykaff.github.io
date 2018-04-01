### fork/join示例篇

时间: 2018年3月21日

根据fork/join思想构造一个并行递归批改试卷的示例（示例程序中用1s代替1小时休眠（拼命批改）进行计算）：

```
package forkjoin;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;

import static java.util.stream.Collectors.toList;

/**
 * fork-join的简单易用
 * 
 * 2018/3/20
 *
 * @author boyka
 */

/**
 * 任务类:批改的一套试卷或试卷的一部分
 */

class Task {
	private String name;  //任务名
	private String level; //困难等级
	private Long useTime; //预计花费时间
	private List<Task> subTaskList; //子任务列表

	public Task(String name, String level, Long useTime, List<Task> subTaskList) {
		this.name = name;
		this.level = level;
		this.useTime = useTime;
		this.subTaskList = subTaskList;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getLevel() {
		return level;
	}

	public void setLevel(String level) {
		this.level = level;
	}

	public Long getUseTime() {
		return useTime;
	}

	public void setUseTime(Long useTime) {
		this.useTime = useTime;
	}

	public List<Task> getSubTaskList() {
		return subTaskList;
	}

	public void setSubTaskList(List<Task> subTaskList) {
		this.subTaskList = subTaskList;
	}
}

/**
 * 任务操作类:进行试卷批改操作
 */
public class ForkJoinTest extends RecursiveTask<Long> {
	private final static String EASY = "easy";
	private final static String DIFFICULT = "difficult";
	private Task task;

	private ForkJoinTest(Task task) {
		this.task = task;
	}

	@Override
	protected Long compute() {
		Long useTimes = 0L;
		//容易，一个同学能够改得完了
		if (EASY.equals(task.getLevel())) {
			useTimes = task.getUseTime();
			try {
				Thread.sleep(task.getUseTime() * 1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		} else {
			//任务量大，将批改试卷任务按任务量细分后让多个童鞋去(工作线程)完成
			if (null != task && null != task.getSubTaskList()) {
				useTimes = task.getSubTaskList()
						.stream()
						.map(t -> new ForkJoinTest(t).fork())
						.collect(toList())
						.stream()
						.mapToLong(t -> t.join())
						.sum();
			}
		}
		return useTimes;
	}

	// 测试
	public static void main(String[] args) throws ExecutionException, InterruptedException {
		long startTime = System.currentTimeMillis();
		ForkJoinPool forkJoinPool = new ForkJoinPool();
		ForkJoinTest mainForkJoin = new ForkJoinTest(ForkJoinTest.initTasks());
		ForkJoinTask<Long> result = forkJoinPool.submit(mainForkJoin);
		System.out.println("BOSS一个人批改用时：" + result.get() + " H");
		System.out.println("多个童鞋一起改用时： " + (System.currentTimeMillis() - startTime) / 1000 + " H");
	}

	//构造简易批改试卷任务树
	private static Task initTasks() {
		//微机
		Task wjTask1 = new Task("微机选择+填空题", EASY, 4L, new ArrayList<>());
		Task wjTask2 = new Task("微机大题", EASY, 3L, null);
		Task wjTask = new Task("算法试卷", DIFFICULT, null, Arrays.asList(wjTask1, wjTask2));
		//网络
		Task wlTask = new Task("网络", EASY, 6L, null);

		//算法
		Task sfTask1 = new Task("算法前3道", EASY, 4L, null);
		Task sfTask2 = new Task("算法后2道", EASY, 4L, null);
		Task sfTask = new Task("算法试卷", DIFFICULT, null, Arrays.asList(sfTask1, sfTask2));

		return new Task("所有试卷", DIFFICULT, null, Arrays.asList(wjTask, wlTask, sfTask));
	}
}
```

测试结果：

```
BOSS一个人批改用时：21 H
多个童鞋一起改用时： 6 H
```

阔以看出，效果很明显，童鞋们帮BOSS大忙啦，BOSS请客吃饭，皆大欢喜。

上面这个例子体现出fork/join框架的一个重要特性，就是将一个大任务拆分成多个小任务并行执行，看上面的任务分配：有一个童鞋3个小时（微机大题）就完成就能够修改分配给自己要修改的试卷啦，然而如果他能帮助那个需要花费6个小时(计算机网络)的童鞋修改一部分还没有改完的试卷，那是不是大家都可以提前改完，提前去吃大餐啦，呼呼，这就可以看成fork/join的另一个重要特性-工作窃取（work-stealing）。

ok, 看了这个示例，效果真是棒，但你是否还有以下疑问：

- ForkJoinPool、ForkJoinTask、RecursiveTask（还有一个ForkJoinWorkerThread工作线程没在代码中体现出来）的关系？
- ForkJoinTest为什么要继承RecursiveTask？
- fork() 和join() 阶段干了些什么事情？
- 工作窃取过程如何完成的？

有疑问，继续看看鄙人陋作fork/join原理浅析篇望能够解惑[fork/join源代码浅析篇](../1/fork-join框架翻译.html>)






