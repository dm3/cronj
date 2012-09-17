# cronj

This is *another* cron-inspired task-scheduling library. I have found many scheduling libraries for clojure:

  - [quartzite](https://github.com/michaelklishin/quartzite)
  - [cron4j](http://www.sauronsoftware.it/projects/cron4j)
  - [clj-cronlike](https://github.com/kognate/clj-cronlike)
  - [at-at](https://github.com/overtone/at-at)
  - [monotony](https://github.com/aredington/monotony)

The first three all follow the cron convention. The "task" (also called a "job") can only be scheduled at whole minute intervals. [at-at](https://github.com/overtone/at-at) has milli-second resolution, but was limited in the number of threads that have to be predetermined. It was good for looking after tasks that did not overlap between calls but not for tasks that may take an arbitarily long time. [monotony](https://github.com/aredington/monotony) uses core.logic, which is something that I am yet to understand.

I needed something that

  - started scheduled tasks with a per-second interval having high system-time accuracy without wasting system resourcs.
  - would spawn as many threads as needed, so that tasks started at earlier intervals could exist along side tasks started at later intervals.
  - an additional design requirement required that task handlers are passed a date-time object, so that the handler itself is aware of the time when it was initiated.

### Installation:
 
In project.clj, add to dependencies:
     
     [cronj "0.2.0"]


### Usage

    (require '[cronj.core :as cj])

    (cj/load-tasks!
     [{:id 1 :desc 1 :handler #(println "Task 1: " %) :tab "0-60/2 * * * * * *"}
      {:id 2 :desc 2 :handler #(println "Task 2: " %) :tab "1-60/2 * * * * * *"}])
    
    (cj/start!)
    
    ;; wait for outputs ......
    > Task 1:  #DateTime 2012-09-17T14:10:24.000+10:00
    > Task 2:  #DateTime 2012-09-17T14:10:25.001+10:00
    > Task 1:  #DateTime 2012-09-17T14:10:26.000+10:00
    > Task 2:  #DateTime 2012-09-17T14:10:27.001+10:00
    > Task 1:  #DateTime 2012-09-17T14:10:28.000+10:00
    
    ;; get bored and stop
    (cj/stop!)
    
    
### Arguments
Additional arguments can be added to the handler through the args option and a function that is takes the arguments

    (cj/unschedule-all-tasks)
    (cj/schedule-task! {:id 3 :desc 3 
                        :handler (fn [dt & {:keys [task-name]}] (println task-name ": " dt)) 
                        :tab "/2 * * * * * *"
                        :args {:task-name "Hello There"}})
    (cj/start!)
    ;; more outputs ......
    > Hello There :  #DateTime 2012-09-17T14:53:10.000+10:00
    > Hello There :  #DateTime 2012-09-17T14:53:12.000+10:00
    > Hello There :  #DateTime 2012-09-17T14:53:14.001+10:00

    ;; stop outputs
    (cj/stop!)


Thats really it... now write your own!

## The Details

Tasks can be added and removed on the fly through the `cronj` library interface and the library will then keep an eye out on the time. At the correct time that a task has been scheduled to start, the task handler will be launched in another thread. The actual polling loop is quite efficient and will only poll a maximum of twice every second with a 1ms timing error. Future improvements to the loop will hope to preempt the time that tasks should start and sleep until it is necessary to wake up.


### Overview:

Cronj is seperated into three basic concepts:

- Tasks (are records that provide information about the task)
      
- A Timesheet (to strictly schedule and unschedule tasks according to a `tab` schedule as well as as provide functionality to easily manipulate groups of tasks.)
      
- A Timekeeper (keeps the time as triggers tasks at the interval that it is scheduled to run)

                                      timesheet
              __...--+----+----+----+----+----+----+----+----+----+----+----+----+
     _..---'""      _|.--"|    |    |    |    |    |    |    |    |    |    |    |
    +-------------+'_+----+----+----+----+----+----+----+----+----+----+----+----+
    | task        |-     /                                              |
    |             |     /                     X                         |
    |    :id      |    /                    XXXXX              timesheet  methods
    |    :desc    |   /                    XXXXXXX          +-------------------------+
    |  ++:handler |  /                    XXXXXXXXX           contains-task? [id]
    | +++:tab       /                        XXX              select-task    [id]
    | || :enabled |/                         XXX              enable-task!   [id]
    +-++----------+                          XXX              disable-task!  [id]
      ||  ,-.                                XXX              trigger-task!  [id]
      |+-(   ) fn[dt & args]                 XXX              list-running-for-task  [id]
      |   `-'                                XXX              kill-all-running-for-task! [id]
     +-------------------------+             XXX              kill-running-for-task! [id tid]
     |  "* 8 /2 7-9 2,3 * *"   |             XXX
     +-------------------------+             XXX              list-all-tasks []
     |  :sec    [:*]           |             XXX              load-tasks!    [v]
     |  :min    [:# 8]         |             XXX              unschedule-all-tasks! []
     |  :hour   [:| 2]         |          XXXXXXXXX           schedule-task!   [id]
     |  :dayw   [:- 7 9]       |        XX         XX         unschedule-task! [id]
     |  :daym   [:# 2] [:# 3]  |      XX             XX
     |  :month  [:*]           |     X      cronj      X
     |  :year   [:*]           |    X                   X
     +-------------------------+    X     :thread       X
                                    X     :last-check   X        timekeeper methods
        cronj function               X    :interval    X    +-------------------------+
        --------------                XX             XX        running?     start!
       At every second.                 XX         XX          stopped?     stop!
       looks at task                      XXXXXXXXX            restart!
       list and triggers
       handler functions
       for each enabled
       task.


### Tasks:

A "task" has the following attributes:

      - "id" and "desc" for meta description of the task
      - "handler", the actual procedure that provides the functionality for a task
      - arguments to the handler can be passed via the optional "args" attribute

A task does not have a concept of when it will run. It only responds when it is asked to run and will keep track of all the instances of the task that are running. It is defined as follows:

    (require '[cronj.task :as ct])
    (require '[clj-time.core :as t])
    (def task-10s (ct/new :10 "10s" 
                         (fn [_] (println "I Last for 10 seconds")  
                         (Thread/sleep 10000))))

And triggered using:

    (ct/exec! task-10s *id*)
                                         
where *id* has to be a unique identifier. In this case, we will use the the datetime. So when:

    (ct/exec! task-10s (t/now))
    ;; => I Last for 10 seconds
    (ct/running task-10s) ;; within ten seconds
    ;; => (#<DateTime 2012-09-17T05:19:36.223Z>)

    
### Long Running Tasks
    
A task may be scheduled to run every 30 seconds, but may take up to a minute to finish. In this case, the multiple threads will be spawned for multiple calls. Below is an example:

    (ct/exec! task-10s (t/now))
    (ct/exec! task-10s (t/now))
    (ct/exec! task-10s (t/now))
    (ct/running task-10s) ;; within ten seconds of the three
    ;; => (#DateTime 2012-09-17T05:21:19.967Z #DateTime 2012-09-17T05:21:21.111Z #DateTime 2012-09-17T05:21:21.949Z)
    
    ;; and then they can be killed individually or together:
    (ct/kill-all task-10s)
    (ct/running task-10s)
    ;; => ()

The symbol `task-10s` actually references a map:

    (println job-10s)
    ;; => {:enabled #[Atom@5148bd9e: true], :running [DynaRec], :args {}, :last-called #[Atom@7e98f9c2: nil], :last-successful #[Atom@6d35707c: nil], :id :10, :desc 10s, :handler #user$fn}

The complete map attributes are described:

    :id                The unique identifier for the task     
    :desc              A description for the task
    :handler           The function that is called using `exec!`
    :enabled           If false, the task will not run
    :args              Additionally arguments to the handler
    :running           All the instances of the task that are still running
    :last-called       The id of the last instance called using `exec!`
    :last-successful   The id of the last instance that finished normally.
    
   
### More Examples

   (cj/schedule-task! {:id "print-date"
             :desc "prints out the date every 5 seconds"
             :handler #'println
             :tab "/5 * * * * * *"})

    (cj/schedule-task! {:id "print-date"
            :desc "prints out the date every 5 seconds between 32 and 60 seconds"
            :handler #'println
            :tab "32-60/5 * * * * * *"})


    (cj/schedule-task! {:id "print-date"
             :desc "prints out the date every 5 seconds on the
                    9th aand 10th minute of every hour on every Friday
                    from June to August between the year 2012 to 2020"
             :handler #'println
             :tab "/5  9,10  * 5 * 6-8 2012-2020"}) 


### Methods
The functions in `cronj.core` should be pretty self evident:

Basic:

    list-all-tasks []                 
    load-tasks!    [v]
    unschedule-all-tasks! []               
    schedule-task!   [id]              
    unschedule-task! [id]             

Controls:

    stopped? []
    running? []
    start!   []
    stop!    []
    restart! []

Task Related:

    contains-task? [id]
    select-task    [id]
    enable-task!   [id]
    disable-task!  [id]
    trigger-task!  [id]
    list-running-for-task  [id]
    kill-all-running-for-task! [id]
    kill-running-for-task! [id tid]

Advanced (not used except in unsual circumstances)

    set-cronj!! [cj]
    new-cronj!! []

## TODO:
- Finish Tests