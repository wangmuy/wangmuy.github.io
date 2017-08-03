---
layout: post
title: "aosp-repo-refs-remotes-m"
date: 2017-04-24 14:30:16 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

# refs/remotes/m 是什么

m 代表 merge, 是 git 仓库里的 .gitconfig 里 branch 的 merge 字段


# refs/remotes/m 怎么来的

```
git_refs.py
R_M = 'refs/remotes/m/'

使用
info.py
Info.findRemoteLocalDiff(project) 列出remote和local的diff

设置/读取
project.py Sync_NetworkHalf()
-> Project._InitMRef() {
  self._InitAnyMRef(R_M + self.manifest.branch) 传参 'refs/remotes/m/'+manifests仓库的currentBranch.merge
  self.revisionId == None: Project的创建在 manifest_xml.py _AddMetaProjectMirror()/_ParseProject() 两处revisionId均为None
    remote=self.GetRemote(self.remote.name)
    dst=remote.ToLocal(self.revisionExpr)
      revisionExpr 顺序: 1.项目的 revision 字段, 2. remote 指定的 revision 字段, 3. default 指定的 revision 字段
      参考: https://gerrit.googlesource.com/git-repo/+/master/docs/manifest-format.txt
    self.bare_git.symbolic_ref('-m', msg, ref, dst) 设置symbolic-ref } 设置的ref log可通过 git reflog refs/remotes/m/rBranch 查看
-> Project._InitAnyMRef(ref) { self.bare_ref.symref(ref) }
-> git_refs.py GitRefs.symref(name) { self._EnsureLoaded(); return self._symref[name] }
  -> GitRefs._EnsureLoaded() { if self._phyref is None or self._NeedUpdate(): self._LoadAll() }
  -> GitRefs._LoadAll() {
    self._ReadPackedRefs() 读取 .git/packed_refs 忽略 # 和 ^ 打头行, foreach 其余每行 ref_id name: self.phyref[name]=ref_id
    self._ReadLoose('refs/') {
      递归读取 .git/refs 目录
      foreach 文件: _ReadLoose1(self, 文件路径path, 相对.git的文件名称name) {
        读取文件 if 'ref: '打头: self._symref[name]=ref_id[5:] else: self._phyref[name]=ref_id
    self._ReadLoose1(os.path.join(self._gitdir, HEAD), HEAD) } 读取 .git/HEAD
```

也就是说，refs/remotes/m/branchName 会在每次sync时候读取，若没有则创建，不需要自己创建并push到remote