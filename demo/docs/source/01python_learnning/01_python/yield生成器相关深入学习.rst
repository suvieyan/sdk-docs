.. _header-n0:

yield生成器相关深入学习
=======================

流畅的Python16章

流畅的Python源代码：https://github.com/hellowac/fluentpythoncode/blob/master/17-futures/countries/flags_threadpool.py

yield已知概念信息：

-  生成器

-  可暂停状态

-  需要next进行触发

-  没有值为StopIteration异常

-  send 发送数据为yield表达式的值

-  throw 调用方抛出异常

-  close 关闭生成器

.. _header-n21:

1.案例一：
----------

1.yield 等同于return 可以直接获取值

2.b = yield a 需要send一个值给b 否则为None.先执行yield 再执行send

.. code:: python

   def simple(a):
       print('--> started a=', a)
       b = yield a  # 客户端代码激活协程才能获取值
       print('--> received b=', b)
       yield b
       c = yield a + b
       print('--> received c=', c)
       yield c


   ret = simple(3)
   print(next(ret))  # print a 和yield a
   print(ret.send(42))  # send值给了b, print b 和yield b
   print(next(ret))  # yield a+b,
   print(next(ret))  # yield a+b,
   # try:
   #     print(ret.send(66))
   # except StopIteration:
   #     pass

   """
   --> started a= 3
   3
   --> received b= 42
   42
   45
   --> received c= None
   None
   """

.. _header-n25:

2.案例2：计算移动平均值
-----------------------

.. code:: python

   """
   计算移动平均值
   """
   def averager():
       total = 0.0
       count = 0
       average = None
       while 1:
           term = yield average
           total += term
           count += 1
           average = total / count


   average = averager()
   next(average)
   print(average.send(10))
   print(average.send(20))
   print(average.send(30))

.. _header-n27:

3.案例3：定义预处理器+处理异常
------------------------------

.. code:: python

   def coroutine(func):
       def inner(*args, **kwargs):
           gen = func(*args, **kwargs)
           next(gen)
           return gen

       return inner


   @coroutine
   def averager():
       total = 0.0
       count = 0
       average = None
       while 1:
           term = yield average
           try:
               total += term
           except TypeError:
               print('数据不合法')
               total += 0
           count += 1
           average = total / count


   average = averager()
   # next(average)
   print(average.send(10))
   print(average.send(20))
   print(average.send(30))
   print(average.send('yan'))
   print(average.send(30))

.. _header-n29:

4.案例4获取协程的返回值
-----------------------

.. code:: python

   # 获取协程的返回值，协程的返回值的赋值给了StopIteration的value属性

   from collections import namedtuple

   Result = namedtuple('Result', 'count average')
   def coroutine(func):
       def inner(*args, **kwargs):
           gen = func(*args, **kwargs)
           next(gen)
           return gen

       return inner


   @coroutine
   def averager():
       total = 0.0
       count = 0
       average = None
       while 1:
           term = yield average
           if term is None:
               break
           total += term
           count += 1
           average = total / count
       return Result(count, average)


   average = averager()
   # next(average)
   print(average.send(10))
   print(average.send(20))
   try:
       print(average.send(None))
   except StopIteration as e:
       print(e.value)  # Result(count=2, average=15.0)

.. _header-n31:

5.yield 和yield from的区别
--------------------------

.. code:: python

   def gen():
       for c in "AB":
           yield c

       for i in range(3):
           yield i

   ret = list(gen())
   print(ret)  # ['A', 'B', 0, 1, 2]

   def gen():
       # yield from 把产出的值直接传递给gen的调用者
       yield from "AB"
       yield from range(3)


   ret = list(gen())
   print(ret)  # ['A', 'B', 0, 1, 2]

   """
   yield from X 
   1,调用iter(X), 获取迭代器。严谨说明把职责委托给子生成器，即X
   2，"AB"以及range(3) 都是子生成器
   3，外部代码可以直接和子生成器交互
   """

.. _header-n33:

6.yieldfrom 计算平均值案例
--------------------------

.. code:: python

   from collections import namedtuple

   Result = namedtuple('Result', 'count average')


   def coroutine(func):
       def inner(*args, **kwargs):
           gen = func(*args, **kwargs)
           next(gen)
           return gen

       return inner


   # @coroutine
   def averager():
       total = 0.0
       count = 0
       average = None
       while 1:
           term = yield
           if term is None:
               break
           total += term
           count += 1
           average = total / count
       return Result(count, average)


   def grouper(results, key):
       while 1:
           results[key] = yield from averager()


   # the client code, a.k.a. the caller
   def main(data):  # 8
       results = {}
       for key, values in data.items():
           group = grouper(results, key)  # 获取一个average实例
           next(group)  # 10
           for value in values:
               group.send(value)  # 把值传递给average实例实例
           group.send(None)  # 当前average实例 实例终止

       # print(results)  # uncomment to debug
       report(results)


   # output report
   def report(results):
       for key, result in sorted(results.items()):
           group, unit = key.split(';')
           print('{:2} {:5} averaging {:.2f}{}'.format(
               result.count, group, result.average, unit))


   data = {'girls;kg':
               [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
           'girls;m':
               [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
           'boys;kg':
               [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
           'boys;m':
               [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
           }

   if __name__ == '__main__':
       main(data)

.. _header-n35:

7.yield from 内部执行
---------------------

1.next子生成器

2,获取子生成器的值

-  捕获close异常

-  捕获throw异常

-  获取值判断是否StopIteration异常

.. _header-n46:

8.生成器做仿真
--------------

.. code:: python

   """
   Taxi simulator
   ==============
   Driving a taxi from the console::
       >>> from taxi_sim import taxi_process
       >>> taxi = taxi_process(ident=13, trips=2, start_time=0)
       >>> next(taxi)
       Event(time=0, proc=13, action='leave garage')
       >>> taxi.send(_.time + 7)
       Event(time=7, proc=13, action='pick up passenger')
       >>> taxi.send(_.time + 23)
       Event(time=30, proc=13, action='drop off passenger')
       >>> taxi.send(_.time + 5)
       Event(time=35, proc=13, action='pick up passenger')
       >>> taxi.send(_.time + 48)
       Event(time=83, proc=13, action='drop off passenger')
       >>> taxi.send(_.time + 1)
       Event(time=84, proc=13, action='going home')
       >>> taxi.send(_.time + 10)
       Traceback (most recent call last):
         File "<stdin>", line 1, in <module>
       StopIteration
   Sample run with two cars, random seed 10. This is a valid doctest::
       >>> main(num_taxis=2, seed=10)
       taxi: 0  Event(time=0, proc=0, action='leave garage')
       taxi: 0  Event(time=5, proc=0, action='pick up passenger')
       taxi: 1     Event(time=5, proc=1, action='leave garage')
       taxi: 1     Event(time=10, proc=1, action='pick up passenger')
       taxi: 1     Event(time=15, proc=1, action='drop off passenger')
       taxi: 0  Event(time=17, proc=0, action='drop off passenger')
       taxi: 1     Event(time=24, proc=1, action='pick up passenger')
       taxi: 0  Event(time=26, proc=0, action='pick up passenger')
       taxi: 0  Event(time=30, proc=0, action='drop off passenger')
       taxi: 0  Event(time=34, proc=0, action='going home')
       taxi: 1     Event(time=46, proc=1, action='drop off passenger')
       taxi: 1     Event(time=48, proc=1, action='pick up passenger')
       taxi: 1     Event(time=110, proc=1, action='drop off passenger')
       taxi: 1     Event(time=139, proc=1, action='pick up passenger')
       taxi: 1     Event(time=140, proc=1, action='drop off passenger')
       taxi: 1     Event(time=150, proc=1, action='going home')
       *** end of events ***
   See longer sample run at the end of this module.
   """

   import random
   import collections
   import queue
   import argparse
   import time

   DEFAULT_NUMBER_OF_TAXIS = 3
   DEFAULT_END_TIME = 180
   SEARCH_DURATION = 5
   TRIP_DURATION = 20
   DEPARTURE_INTERVAL = 5

   Event = collections.namedtuple('Event', 'time proc action')


   # BEGIN TAXI_PROCESS
   def taxi_process(ident, trips, start_time=0):  # <1>
       """
       每次改变状态时创建事件，把控制权让给仿真器
       :param ident:出租车编号
       :param trips:出租车回家之前的行程数量
       :param start_time:出租车离开车库时间
       :return:
       """
       time = yield Event(start_time, ident, 'leave garage')  # 第一个event是leave garage
       for i in range(trips):  # <3>
           time = yield Event(time, ident, 'pick up passenger')  # <4>
           time = yield Event(time, ident, 'drop off passenger')  # <5>

       yield Event(time, ident, 'going home')  # <6>
       # 出租车进程结束
   # END TAXI_PROCESS


   # BEGIN TAXI_SIMULATOR
   class Simulator:

       def __init__(self, procs_map):
           self.events = queue.PriorityQueue()
           self.procs = dict(procs_map)  # 3个生成器

       def run(self, end_time):  # <1>
           """Schedule and display events until time is up"""
           # schedule the first event for each cab
           for _, proc in sorted(self.procs.items()):  # <2>
               first_event = next(proc)  # 激活每一个生成器
               self.events.put(first_event)  # 把生成器放入events当中
           print(111111111, self.events._qsize())

           # main loop of the simulation
           sim_time = 0  # <5>
           while sim_time < end_time:  # <6>
               if self.events.empty():  # <7>
                   print('*** end of events ***')
                   break

               current_event = self.events.get()  # 获取一个event
               # sim_time 时间；proc_id taxi编号；previous_action 行动信息
               sim_time, proc_id, previous_action = current_event  # <9>
               print('taxi:', proc_id, proc_id * '   ', current_event)  # <10>
               active_proc = self.procs[proc_id]  # 获取编号对应的生成器taxi
               next_time = sim_time + compute_duration(previous_action)  # <12>
               try:
                   next_event = active_proc.send(next_time)  # <13>
               except StopIteration:
                   del self.procs[proc_id]  # <14>
               else:
                   self.events.put(next_event)  # <15>
           else:  # <16>
               msg = '*** end of simulation time: {} events pending ***'
               print(msg.format(self.events.qsize()))
   # END TAXI_SIMULATOR


   def compute_duration(previous_action):
       """Compute action duration using exponential distribution"""
       if previous_action in ['leave garage', 'drop off passenger']:
           # new state is prowling
           interval = SEARCH_DURATION
       elif previous_action == 'pick up passenger':
           # new state is trip
           interval = TRIP_DURATION
       elif previous_action == 'going home':
           interval = 1
       else:
           raise ValueError('Unknown previous_action: %s' % previous_action)
       return int(random.expovariate(1/interval)) + 1


   def main(end_time=DEFAULT_END_TIME, num_taxis=DEFAULT_NUMBER_OF_TAXIS,
            seed=None):
       """Initialize random generator, build procs and run simulation"""
       if seed is not None:
           random.seed(seed)  # get reproducible results

       taxis = {i: taxi_process(i, (i+1)*2, i*DEPARTURE_INTERVAL)
                for i in range(num_taxis)}
       sim = Simulator(taxis)
       sim.run(end_time)


   if __name__ == '__main__':

       parser = argparse.ArgumentParser(
                           description='Taxi fleet simulator.')
       parser.add_argument('-e', '--end-time', type=int,
                           default=DEFAULT_END_TIME,
                           help='simulation end time; default = %s'
                           % DEFAULT_END_TIME)
       parser.add_argument('-t', '--taxis', type=int,
                           default=DEFAULT_NUMBER_OF_TAXIS,
                           help='number of taxis running; default = %s'
                           % DEFAULT_NUMBER_OF_TAXIS)
       parser.add_argument('-s', '--seed', type=int, default=None,
                           help='random generator seed (for testing)')

       args = parser.parse_args()
       main(args.end_time, args.taxis, args.seed)


   """
   Sample run from the command line, seed=3, maximum elapsed time=120::
   # BEGIN TAXI_SAMPLE_RUN
   $ python3 taxi_sim.py -s 3 -e 120
   taxi: 0  Event(time=0, proc=0, action='leave garage')
   taxi: 0  Event(time=2, proc=0, action='pick up passenger')
   taxi: 1     Event(time=5, proc=1, action='leave garage')
   taxi: 1     Event(time=8, proc=1, action='pick up passenger')
   taxi: 2        Event(time=10, proc=2, action='leave garage')
   taxi: 2        Event(time=15, proc=2, action='pick up passenger')
   taxi: 2        Event(time=17, proc=2, action='drop off passenger')
   taxi: 0  Event(time=18, proc=0, action='drop off passenger')
   taxi: 2        Event(time=18, proc=2, action='pick up passenger')
   taxi: 2        Event(time=25, proc=2, action='drop off passenger')
   taxi: 1     Event(time=27, proc=1, action='drop off passenger')
   taxi: 2        Event(time=27, proc=2, action='pick up passenger')
   taxi: 0  Event(time=28, proc=0, action='pick up passenger')
   taxi: 2        Event(time=40, proc=2, action='drop off passenger')
   taxi: 2        Event(time=44, proc=2, action='pick up passenger')
   taxi: 1     Event(time=55, proc=1, action='pick up passenger')
   taxi: 1     Event(time=59, proc=1, action='drop off passenger')
   taxi: 0  Event(time=65, proc=0, action='drop off passenger')
   taxi: 1     Event(time=65, proc=1, action='pick up passenger')
   taxi: 2        Event(time=65, proc=2, action='drop off passenger')
   taxi: 2        Event(time=72, proc=2, action='pick up passenger')
   taxi: 0  Event(time=76, proc=0, action='going home')
   taxi: 1     Event(time=80, proc=1, action='drop off passenger')
   taxi: 1     Event(time=88, proc=1, action='pick up passenger')
   taxi: 2        Event(time=95, proc=2, action='drop off passenger')
   taxi: 2        Event(time=97, proc=2, action='pick up passenger')
   taxi: 2        Event(time=98, proc=2, action='drop off passenger')
   taxi: 1     Event(time=106, proc=1, action='drop off passenger')
   taxi: 2        Event(time=109, proc=2, action='going home')
   taxi: 1     Event(time=110, proc=1, action='going home')
   *** end of events ***
   # END TAXI_SAMPLE_RUN
   """
