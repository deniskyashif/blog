---
title: "Task Management using Emacs and org-mode"
date: 2023-08-27T09:43:13+03:00
draft: true
summary: "Desribing my org-mode workflow."
editLink: "https://github.com/deniskyashif/blog/blob/master/content/"
tags: ["emacs", "productivity"]
---

During my daily work I need to be simultaneously on top of several projects. I do technical groundwork, plan, corrdinate, follow-up with people. This is not always easy as the context switching can be quite exhausting. I prefer to use my memory bandwidth sparingly only when really needed.

Taks management is difficult to get right. One can easily overdo it, causing it to become a chore and increase one's workload. One can also underdo it, which then renders it useless.

My task management is has two driving factors:

- Easy Read - I should quickly get the information I need and never miss anything.
- Easy Write - maintaining my tasks should not cause me an extra effort.

It took me a while to come up with a process matching the above properties. I tried several tools - some had lots of features but required too much effort, others were simple but harder to navigate for larger workloads. I needed to setup my workflow using a tool and it has to feel _just right_ and I found it in Emacs and Org-mode.

## Not everything should be a task

It's important to keep this in mind before diving into the workflow. Sometimes we just need to "do the thing" and move on. I add items to my task list only when I need to, otherwise I would end up with a cluttered agenda of mostly useless items and lose the "Easy Write" property.

## Org-mode config

I have a single org file on my computer `~/org/tasks.org`.

```lisp
(setq org-agenda-files '("~/org"))
```

My tasks have three states - TODO, PROG, DONE. I delete cancelled tasks and archive the DONE ones (more on that later).

```lisp
(setq org-todo-keywords
      '((sequence "TODO(t)" "PROG(p)" "DONE(d)")))
```

## Structure

```
|-- Root
   |-- [Project]      :Tag:
      |-- [Task]
   ...
```

This structure allows me to expand/collapse and easily navigate through projects. I have a single root - Tasks, and General/Project-based nodes. I tag each of those nodes so that the tasks below can inherit it. That comes handy when searching/filtering my agenda. My tree rarely gets deeper that three levels. Sometimes I break down a task into subtasks using bullets. In general, however, I try to keep the structure flat.

Here is an example:

[TASKS.ORG gif]

Just like not everything deserves a to be a Task, not every Task deserves to have a deadline. I set deadlines only when sometihng is time critical. I set scheduled dates only for things which take more thay a day or two to complete. Not all tasks have to be completed either. We have shifting priorities so it's OK for a task to remain in the list for as long as it's needed.

## Navigation

I have a custom agenda view bound to the `n` key where I group my tasks by their state. It shows me the weekly distribution by deadline/schedule and a global list of tasks groped by their state.

```lisp
  ;; Customized view for the daily workflow. (Command: ", a n")
  (setq org-agenda-custom-commands
        '(("n" "Weekly Agenda View with All the Tasks"
           ((agenda "" nil)
            (todo "PROG" nil)
            (todo "TODO" nil)
            (todo "DONE" nil))
           nil)))

(setq org-todo-keyword-faces
      '(("PROG" . (:foreground "yellow" :weight bold))))

;; Hide the deadline prewarning prior to scheduled date.
(setq org-agenda-skip-deadline-prewarning-if-scheduled 'pre-scheduled)
```

### Show my custom weekly agenda view

`(org-agenda) n`

In my weekly agenda buffer I:

- Switch to next/previous week by pressing `f`/`b` respectively.
- Run the `(org-agenda-filter)` (bound to `\`) command and input `+TagName` to see only the tasks for a specific project (or General tasks).
- Change a task state by pressing `t`.
- Edit a task by placing the caret on it and pressing `[Tab]`.

[WEEKLY AGENDA NAV GIF]

### List all the TOOD entries

`(org-agenda) t`

[TODO LIST AGENDA NAV GIF]

Here I filter my tasks by state.

## Archiving

## Concluison

There are still areas of org-mode and Emacs which I'm yet to learn. This wokrflow is what I've come up with so far and has been working well for my needs. The Agenda view is one of the screens I keep open most of the time and the task list is what I usually start my days with. Note that I'm using Spacemacs so the ergonomics might differ on other Emacs distributions.
