---
layout:     post
title:     Django配置celery执行动态周期任务
subtitle:     蓝鲸PaaS平台app开发经验分享（1）
date:       2019-03-20
author:     贺洋
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 蓝鲸
    - 自动化运维
    - PaaS
    - 元鼎
---
> 本文主要介绍蓝鲸SaaS开发场景中，celery定时任务的简单使用

# 蓝鲸PaaS介绍

腾讯蓝鲸智云是一套基于PaaS的技术解决方案，提供了完善的前后台开发框架、调度引擎、公共组件等模块，帮助业务的产品和技术人员快速构建低成本、免运维的支撑工具和运营系统。

PaaS平台不仅将应用服务的运行和开发环境作为一种服务提供给开发者用户，更为开发者用户提供了高效便捷的开发服务，如：组件系统，统一登录，权限管理，后台框架，MagicBox，桌面/工作台等。

![paas](https://docs.bk.tencent.com/paas/assets/image002.png)

PaaS平台提供支持多语言的开发框架，助力运维人员能基于平台之上以自己擅长的技术语言（Python、php 等）开发运维自动化工具。

通过了解PaaS的设计理念，运维人员能够基于蓝鲸的PaaS平台，从零开始进行SaaS应用的实战开发，快速构建企业运维/运营系统，提升企业自动化水平。

# 开发背景

之前在一个银行自动化运维项目中，客户希望我们在蓝鲸PaaS上开发一个数据库巡检SaaS。具体需求如下：

> 为了保障数据库正常运行，保证数据的安全性、完整性和可用性，需要开发一个自动化巡检工具，代替原来的人工数据库巡检。并且巡检周期窗口分为日巡检、周巡检、月巡检、半年度巡检四类：

 * 日巡检维护指每日按计划进行的巡检维护活动，以检查数据库运行状态、数据库备份状态和告警错误为主要内容。
 * 周巡检维护指按一周为周期，在每周指定日按计划进行的巡检维护活动，它的工作内容是在日巡检维护工作内容的基础上添加数据库对象检查、安全性检查等内容组成。
 * 月巡检维护指按一月为周期，在每月指定日按计划进行的巡检维护活动，它的工作内容是在周巡检维护工作内容的基础上添加系统参数配置检查、硬件与系统平台运行状态检查等内容组成。
 * 年度巡检维护指按半年或者一年为周期，在指定日按计划进行的巡检维护活动，它的工作内容是在月巡检维护工作内容的基础上添加数据库性能诊断检查组成。

巡检实现方式分为两种：

* 立即巡检
用户首先选择某一业务下对应的目标主机，以及需要巡检的数据库实例（支持多选），设置数据采样区间（当前时间之前的任意时间段）。
![巡检参数设置](http://img.yuandingit.com/20190321173115761.png)

点击立即巡检按钮，等待数秒钟，巡检完成。点击查看详情，导出报告。

![巡检报告列表](http://img.yuandingit.com/20190322151735624.png)


* 定时巡检
用户可以根据需求设置每天、每周、每月来执行巡检任务。如果选择每周执行，用户需要配置某业务下面主机，数据库实例，巡检频率，巡检时长（任意天数）、执行时间（每周某一天的某时某分某秒），如下图：

![定时巡检设置](http://img.yuandingit.com/20190128155328189.png)

# 定时巡检代码实现

> celery是一个基于python开发的简单、灵活且可靠的分布式任务队列框架，支持使用任务队列的方式在分布式的机器/进程/线程上执行任务调度。采用典型的生产者-消费者模型，主要由三部分组成：
> 1. 消息队列broker：broker实际上就是一个MQ队列服务，可以使用redis、rabbitmq等作为broker
> 2. 处理任务的消费者workers：broker通知worker队列中有任务，worker去队列中取出任务执行，每一个worker就是一个进程
> 3. 存储结果的backend：执行结果存储在backend，默认也会存储在broker使用的MQ队列服务中，也可以单独配置用何种服务做backend

![celery_01](http://img.yuandingit.com/celery_01.png)

针对以上需求，平时我们开发时使用periodic_task装饰器，程序启动后自动执行周期任务：

```
@periodic_task(run_every=crontab(minute='*/5', hour='*', day_of_week="*"))
def get_time():
    """
    celery 周期任务示例

    run_every=crontab(minute='*/5', hour='*', day_of_week="*")：每 5 分钟执行一次任务
    """
    now = datetime.datetime.now()
    logger.error(u"celery 周期任务调用成功，当前时间：{}".format(now))

```

> crontab()实例化的时候没设置任何参数，都是使用默认值。crontab一共有7个参数，常用有5个参数分别为：
minute：分钟，范围0-59
hour：小时，范围0-23
day_of_week：星期几，范围0-6。以星期天为开始，即0为星期天。这个星期几还可以使用英文缩写表示，例如“sun”表示星期天
day_of_month：每月第几号，范围1-31
month_of_year：月份，范围1-12

以上代码实现方式有个弊端：

1. 如果巡检频率选择“周”，开发人员需要根据巡检执行时间（周几），计算出当前日期，结合巡检时长来计算数据采样区间，除了代码比较长以外，可能会因为考虑不周导致时间计算出现误差。
2. 定时巡检任务一旦运行，后续无法直接取消（需要到数据库里手动删除）

举例来说，假如客户选择每周三早上8点执行任务，采样区间为3天。假如首次10月1日8:00执行任务，触发定时任务获取9月28日8:00-10月1日8:00之间的数据；然后再次执行时间为10月8日，代码逻辑上需要考虑和计算各种情况。

最终，通过以下方式解决：
模板函数提前开发完成，加上@task()装饰器：

```
@task()
def auto_iip(**kwargs):
	    logger.error(kwargs)
	    '此处写逻辑代码'
    
```
测试每分钟执行一次，启动工程，启动celery，调用下面函数，OK，等待1分钟，sucess！

```
from djcelery.models import PeriodicTask, CrontabSchedule
from djcelery.schedulers import ModelEntry, DatabaseScheduler
def test_celery_task(date_data):
    crontab= CrontabSchedule.objects.create(
        hour='*',
        minute='*/1',
        day_of_week='*',
        day_of_month='*',
        month_of_year="*"
    )
    schedule = crontab.schedule

    create_or_update_task = DatabaseScheduler.create_or_update_task
    #'home_application.celery_tasks.auto_iip' home模块下的task。
    task_template='home_application.celery_tasks.auto_iip'
    #task_name自定义，不能重复。
    task_name = 'test'
    schedule_dict = {
        'schedule': schedule,
        'args': [],
        'kwargs': data,
        'task': task_template,
        'enabled': 1
    }
    create_or_update_task(task_name, **schedule_dict)
```

定时任务停止，直接根据task_name进行删除

```
def delete_celery_task(task_name):
    DatabaseScheduler.delete_task(task_name)
```

ok！大功告成。