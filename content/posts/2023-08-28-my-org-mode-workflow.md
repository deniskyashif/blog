---
title: "Task Management using Emacs and Org-mode"
date: 2023-08-28T08:23:13+03:00
draft: false
summary: "How I organize my work in plain text."
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2023-08-28-my-org-mode-workflow.md"
tags: ["emacs", "productivity"]
---

During my daily work, I must be on top of several projects. I do the technical groundwork, plan, coordinate, and follow up with people. That is not always easy as the context switching becomes quite exhausting. I've become protective of my memory bandwidth. I use it sparingly, only when necessary, and in areas where I can bring the most value. Hence, I don't try to memorize my work agenda. A plain text file does it for me instead.

Task management is hard to get right. One can easily overdo it, causing it to become a chore and increase one's workload. One can also underdo it, which then renders it useless.

My task management had to have two essential properties.

- __Easy Writing__ - Maintaining my tasks should not cause me extra effort.
- __Easy Reading__ - I should quickly get the needed information and never miss anything.

__After all, the task management has to work for us, not the other way around.__

It took me a while to find a process matching the above properties. I tried several tools. Some had many features, but I found them hard to work with. Others were simple but harder to navigate for larger workloads. I needed to set up my workflow using a tool, and it had to be __just right__. I found it in Emacs and its [Org-mode](https://orgmode.org).

## Not everything deservers a place on the Task list

It's important to keep this in mind before diving into the workflow. Sometimes, we only need to "do the thing" and move on. I add items to my task list sparingly. Otherwise I'd end up with a cluttered agenda with mostly useless information.

## My Org-mode config

I keep everything in a single file `~/org/tasks.org`.

```lisp
(setq org-agenda-files '("~/org"))
```

My tasks have three states - `[TODO] -> [PROG] -> [DONE]`. I delete the cancelled tasks and archive the DONE ones (more on that later).

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

This structure allows me to expand/collapse and easily navigate through projects. I have a single root - Tasks and General/Project-based nodes. I tag each of those nodes so that the tasks below can inherit it. That comes handy when searching/filtering my agenda. My tree rarely gets deeper that three levels. Sometimes, I break down a task into subtasks using bullets. In general, however, I try to keep the structure flat.

Here is an example:

<div>
  <video width="100%" controls src="/images/posts/2023-08-28-my-org-mode-workflow/tasks_org.mov" />
</div>
<br />

Just like not everything deserves a to be a Task not every Task deserves to have a deadline. I set deadlines only when sometihng is time critical. I set scheduled dates only for things which take more thay a day or two to complete. Not all tasks have to be DONE either. We have shifting priorities, so it’s OK for a Task to remain in the list for as long as needed.

## Navigation and Search

My custom agenda view shows me the weekly distribution by deadline/schedule and a global list of tasks groped by their state.

```elisp
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

### Show my custom weekly agenda view `(org-agenda) n`

Most of the work I do in thus buffer falls down to:

- Checking my Tasks for the day/week using the `d`/`w` keys.
- Checking the tasks for a project by  using `\` and input `+{TagName}`
- Updating a Task's state by pressing `t`.
- Editing a Task by placing the caret on it and pressing `[Tab]`.
- Switching to next/previous week by pressing `f`/`b` respectively.

<div>
  <video width="100%" controls src="/images/posts/2023-08-28-my-org-mode-workflow/agenda-view-week.mov" />
</div>
<br />

### List all the TOOD entries `(org-agenda) t`

Here I filter my tasks by state.

<div>
  <video width="100%" controls src="/images/posts/2023-08-28-my-org-mode-workflow/agenda-view-state.mov" />
</div>
<br />

Org-mode has plenty more [ways to search and navigate](https://orgmode.org/worg/org-tutorials/advanced-searching.html), but the above are the ones I use most of the time.

## Archiving

I archive my completed Tasks by pressing `$` when on a task in the Agenda buffer.

<div>
  <video width="100%" controls src="/images/posts/2023-08-28-my-org-mode-workflow/archive.mov" />
</div>

## Concluison

There are still areas of Org-mode and Emacs which I'm yet to explore. This wokrflow, however, has been working well for me. The Agenda view is one of the screens I keep open most of the time, and the Task list is what I usually start my days with. I'm using Spacemacs, so the ergonomics of this workflow might differ in other distributions.
