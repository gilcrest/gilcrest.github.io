---
title: "Apex 5 - Minimize Left Sidebar Navigation by Default"
date: 2015-10-15T20:38:00-04:00
description: Oracle Apex Minimize Sidebar
menu:
  sidebar:
    name: Apex 5 - Minimize Left Sidebar
    identifier: apex5-minimize-sidebar
    parent: archive
    weight: 800
---

You may find yourself in a situation where you proudly unveil your upgraded-to-Universal Theme app to your user-base after working on it for months with your beautiful wide screen monitor (page designer pretty much mandates this) and your user-base is not quite as happy as you thought they'd be as they don't all have super wide screens and they, for some reason, have low-resolution monitors (like 1280 x 768 low).  The users may think your super cool left sidebar navigation, which is maximized by default, is neat, but they'd actually prefer their space back...  If you should find yourself in this situation, there is an easy answer!  By default, always have that left sidebar nav minimized (I actually prefer this anyway).  I turned to the Oracle APEX forum recently for the best solution to do this and forum-master, fac586 gave me a simple solution I thought I'd share for others who may need/want to do this.  I'd actually love to see a declarative way of doing this in a future release of APEX (I'll log a feature request), but until then...

From fac586:
The simplest way to do this is with a dynamic action that simulates clicking on the button when the page is loaded with the menu expanded. Create a dynamic action as follows:

```javascript
Event: Page Load
Condition: JavaScript Expression
Value: $('#t_Button_navControl').attr('aria-expanded') === "true"

True Action

Action: Execute JavaScript Code
Fire On Page Load: No
Code: $('#t_Button_navControl').click();
```

If you want to do this on every page of the application, simply create a global page (unless you already have one) and use the above configuration.  You'll need to use a condition so that this event does not fire on public pages (login and error pages, for example)
