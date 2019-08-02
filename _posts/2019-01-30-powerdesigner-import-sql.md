---
layout: post
title:  【转】PowerDesigner导入SQL注释丢失问题解决
date:   2019-01-30 13:19:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 操作步骤

![操作步骤1](https://upload-images.jianshu.io/upload_images/845143-36617a92ec37ad10.png?jianshufrom=true)

![操作步骤2](https://upload-images.jianshu.io/upload_images/845143-2eb0b515f054b21e.png?jianshufrom=true)

放入的代码：

```
Option Explicit
ValidationMode = True
InteractiveMode = im_Batch

Dim mdl '   the   current   model

'   get   the   current   active   model
Set   mdl   =   ActiveModel
If   (mdl   Is   Nothing)   Then
    MsgBox   "There is no current Model"
ElseIf Not mdl.IsKindOf(PdPDM.cls_Model)   Then
    MsgBox   "The current model is not an Physical Data model."
Else
    ProcessFolder   mdl
End If

Private   sub   ProcessFolder(folder)
On Error Resume Next
      Dim   Tab   'running     table
       for   each   Tab   in   folder.tables
             if   not   tab.isShortcut   then
                  tab.name   =   tab.comment
                  Dim   col   '   running   column
                   for   each   col   in   tab.columns
                   if col.comment="" then
                   else
                        col.name=   col.comment
                   end if
                  next
             end   if
      next

      Dim   view   'running   view
       for   each   view   in   folder.Views
             if   not   view.isShortcut   then 
                  view.name   =   view.comment
             end   if
      next

      '   go   into   the   sub-packages
      Dim   f   '   running   folder
      For   Each   f   In   folder.Packages
             if   not   f.IsShortcut   then
                 ProcessFolder   f
             end   if
      Next
end   sub
```

## 转载链接

[PowerDesigner导入mysql文件注释丢失问题解决
](https://blog.csdn.net/kun_zai/article/details/78722571)