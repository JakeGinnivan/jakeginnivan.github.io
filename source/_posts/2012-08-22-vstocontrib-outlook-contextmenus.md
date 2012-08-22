---
layout: post
title: VSTOContrib Context Menus
metaTitle: New post
description: An issue for VSTO Contrib was submitted, it was hard to wire up context menus, this is now fixed
revised: 2012-08-22
date: 2012-08-22
categories: [vsto-contrib]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vstocontrib-outlook-contextmenus/
summary: | 
  

---
So the scenario was that I want to show additional menu items when right clicking on an email.

    <contextMenus>
      <contextMenu idMso="ContextMenuMailItem">
        <button id="MyContextMenuMailItem"
                label="Perform the action..."
                onAction="EmailItems"
                imageMso="Forward"/>
      </contextMenu>
      <contextMenu idMso="ContextMenuMultipleItems">
        <button id="MyContextMenuMultipleItems"
            label="Perform the action on multiple items..."
            onAction="EmailItems"
            imageMso="Forward"/>
      </contextMenu>
    </contextMenus>

In the current release, the EmailItems callback would never be called. This is because of the way Outlook handles selections and context menus. VSTOContrib now correctly routes the callbacks to the correct viewmodel.

Now you can simply add the callback to your viewmodel like this:

    [OutlookRibbonViewModel(OutlookRibbonType.OutlookExplorer)]
    public class ExplorerViewModel : OfficeViewModelBase, IRibbonViewModel
    {
        Folder context;
        Explorer explorer;

        public IRibbonUI RibbonUi { get; set; }
        public void Initialised(object context)
        {
            this.context = (Folder) context;
        }

        public void EmailItems(IRibbonControl control)
        {
            if (explorer.Selection.Count == 1)
            {
                var title = ((MailItem) explorer.Selection[1]).Subject;
            }
        }

        public void CurrentViewChanged(object currentView)
        {
            explorer = (Explorer) currentView;
        }

        public void Cleanup()
        {
        }
    }

And you will get the EmailItems callback as you would expect. I also recommend putting a getVisible callback to hide the menu item if the selection does not contain mail items for instance.

Hope this helps someone.