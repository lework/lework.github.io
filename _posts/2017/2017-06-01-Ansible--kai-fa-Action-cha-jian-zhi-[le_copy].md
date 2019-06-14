---
layout: post
title: "Ansible 开发Action插件之【le_copy】"
date: "2017-06-01 20:43:44"
categories: Ansible
excerpt: "用于构造action 类 类名需为ActionModule，需从ActionBase继承。类中必须要有run方法。 本次实现的action需求 ..."
auth: lework
---
* content
{:toc}

## 用于构造action 类
---
```
from ansible.plugins.action import ActionBase

class ActionModule(ActionBase):
    def run(self, tmp=None, task_vars=None):
	
        result = super(ActionModule, self).run(tmp, task_vars)
		
        return result
```
类名需为ActionModule，需从ActionBase继承。类中必须要有run方法。


## 本次实现的action需求
---
- 将本地文件拷贝到远程主机


## 创建action
---
在playbook目录下新建action_plugins目录，在action_plugins目录下新建le_copy.py文件。
```
[root@master ansible]# cat action_plugins/le_copy.py
# coding=utf-8

from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

import json
import os
import stat
import tempfile

from ansible.constants import mk_boolean as boolean
from ansible.errors import AnsibleError, AnsibleFileNotFound
from ansible.module_utils._text import to_bytes, to_native, to_text
from ansible.plugins.action import ActionBase
from ansible.utils.hashing import checksum


class ActionModule(ActionBase):

	def run(self, tmp=None, task_vars=None):
		''' handler for file transfer operations '''
		if task_vars is None:
			task_vars = dict()
		# 执行父类的run方法
		result = super(ActionModule, self).run(tmp, task_vars)

		if result.get('skipped'):
			return result

		# 获取参数
		source  = self._task.args.get('src', None)
		dest    = self._task.args.get('dest', None)
		force   = boolean(self._task.args.get('force', 'yes'))
		remote_src = boolean(self._task.args.get('remote_src', False))

		# 判定参数
		result['failed'] = True
		if source is None or dest is None:
			result['msg'] = "src and dest are required"
		elif source is not None and source.endswith("/"):
			result['msg'] = "src must be a file"
		else:
			del result['failed']

		if result.get('failed'):
			return result

		# 如果copy动作在远端执行，直接返回
		if remote_src:
			result.update(self._execute_module(task_vars=task_vars))
			return result

		# 找到source的路径地址
		try:
			source = self._find_needle('files', source)
		except AnsibleError as e:
			result['failed'] = True
			result['msg'] = to_text(e)
			return result

		# 判断是否是目录，如果是跳出返回
		if os.path.isdir(to_bytes(source, errors='surrogate_or_strict')):
			result['failed'] = True
			result['msg'] = to_text(u'不支持目录')
			return result

		changed = False
		module_return = dict(changed=False)

		# 创建临时目录
		if tmp is None or "-tmp-" not in tmp:
			tmp = self._make_tmp_path()

		#  获取本地文件，不存在抛出异常
		try:
			source_full = self._loader.get_real_file(source)
			source_rel = os.path.basename(source)
		except AnsibleFileNotFound as e:
			result['failed'] = True
			result['msg'] = "could not find src=%s, %s" % (source_full, e)
			self._remove_tmp_path(tmp)
			return result


		# 获取远程文件信息
		if self._connection._shell.path_has_trailing_slash(dest):
			dest_file = self._connection._shell.join_path(dest, source_rel)
		else:
			dest_file = self._connection._shell.join_path(dest)

		dest_status = self._execute_remote_stat(dest_file, all_vars=task_vars, follow=False, tmp=tmp, checksum=force)

		# 如果是目录，则返回
		if dest_status['exists'] and dest_status['isdir']:
		   self._remove_tmp_path(tmp)
		   result['failed'] = True
		   result['msg'] = "can not use content with a dir as dest"
		   return result

		# 如果存在，但force为false。则返回
		if dest_status['exists'] and not force:
		  return result

		# 定义拷贝到远程的文件路径
		tmp_src = self._connection._shell.join_path(tmp, 'source')

		# 传送文件

		remote_path = None
		remote_path = self._transfer_file(source_full, tmp_src)

		# 确保我们的文件具有执行权限
		if remote_path:
			self._fixup_perms2((tmp, remote_path))

		# 运行remote_copy 模块
		new_module_args = self._task.args.copy()
		new_module_args.update(
			dict(
				src=tmp_src,
				dest=dest,
				original_basename=source_rel,
			)
		)

		module_return = self._execute_module(module_name='le_copy',
			module_args=new_module_args, task_vars=task_vars,
			tmp=tmp)

		# 判断运行结果
		if module_return.get('failed'):
			result.update(module_return)
			return result
		if module_return.get('changed'):
			changed = True

		if module_return:
		   result.update(module_return)
		else:
		   result.update(dict(dest=dest, src=source, changed=changed))

		# 清理临时文件
		self._remove_tmp_path(tmp)
		
		# 返回结果
		return result
```

action只能在本地执行，所以目标服务器上的操作是要交给modules来做。下面modules是用来把action传送的文件移动到目标位置。

## 创建module
---

在playbook目录下新建library目录，在library目录下新建le_copy.py文件。

> 注：module和action plugin的名称要一致。

```
[root@master ansible]# cat  library/le_copy.py
#!/usr/bin/python
# coding: utf-8

ANSIBLE_METADATA = {'metadata_version': '1.0',
					'status': ['preview'],
					'supported_by': 'community'}

DOCUMENTATION = '''
---
module: le_copy
version_added: "2.3"
short_description: Copy a C(file) to  remote host
description:
	 - The C(le_copy) module copies a file to remote host from a given source to a provided destination.
options:
  src:
	description:
	  - Path to a file on the source file to remote host
	required: true
  dest:
	description:
	  - Path to the destination on the remote host for the copy
	required: true
  force:
	description:
	  - the default is C(yes), which will replace the remote file when contents
		are different than the source. If C(no), the file will only be transferred
		if the destination does not exist.
	required: false
	choices: [ "yes", "no" ]
	default: "yes"
  remote_src:
	description:
	  - If False, it will search for src at originating/master machine, if True it will go to the remote/target machine for the src. Default is False.
	  - Currently remote_src does not support recursive copying.
	choices: [ "True", "False" ]
	required: false
	default: "False"

author:
	- "Lework"
'''

EXAMPLES = '''
# Example from Ansible Playbooks
- name: copy a config C(file)
  le_copy:
	src: /etc/herp/derp.conf
	dest: /root/herp-derp.conf
'''

RETURN = '''
src:
	description: source file used for the copy
	returned: success
	type: string
	sample: "/path/to/file.name"
dest:
	description: destination of the copy
	returned: success
	type: string
	sample: "/path/to/destination.file"
checksum:
	description: sha1 checksum of the file after running copy
	returned: success
	type: string
	sample: "6e642bb8dd5c2e027bf21dd923337cbb4214f827"
gid:
	description: group id of the file, after execution
	returned: success
	type: int
	sample: 100
group:
	description: group of the file, after execution
	returned: success
	type: string
	sample: "httpd"
owner:
	description: owner of the file, after execution
	returned: success
	type: string
	sample: "httpd"
uid:
	description: owner id of the file, after execution
	returned: success
	type: int
	sample: 100
mode:
	description: permissions of the target, after execution
	returned: success
	type: string
	sample: "0644"
size:
	description: size of the target, after execution
	returned: success
	type: int
	sample: 1220
state:
	description: C(state) of the target, after execution
	returned: success
	type: string
	sample: "file"
'''

import os
import shutil


from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils._text import to_bytes, to_native
from ansible.module_utils.pycompat24 import get_exception

def main():
	# 定义modules需要的参数
	module = AnsibleModule(
		argument_spec=dict(
			src=dict(required=True, type='path'),
			dest=dict(required=True, type='path'),
			force=dict(default=True, type='bool'),
			original_basename=dict(required=False),
			remote_src=dict(required=False, type='bool')
		),
		supports_check_mode=True,
	)

	# 获取modules的参数
	src = module.params['src']
	dest = module.params['dest']
	b_src = to_bytes(src, errors='surrogate_or_strict')
	b_dest = to_bytes(dest, errors='surrogate_or_strict')
	force = module.params['force']
	remote_src = module.params['remote_src']
	original_basename = module.params.get('original_basename', None)

	# 判断参数是否合规
	if not os.path.exists(b_src):
		module.fail_json(msg="Source %s not found" % (src))
	if not os.access(b_src, os.R_OK):
		module.fail_json(msg="Source %s not readable" % (src))
	if os.path.isdir(b_src):
		module.fail_json(msg="Remote copy does not support recursive copy of directory: %s" % (src))

	# 获取文件的sha1
	checksum_src = module.sha1(src)
	checksum_dest = None

	changed = False

	# 确定dest文件路径
	if original_basename and dest.endswith(os.sep):
		dest = os.path.join(dest, original_basename)
		b_dest = to_bytes(dest, errors='surrogate_or_strict')

	# 判断目标文件是否存在
	if os.path.exists(b_dest):
		if not force:
			module.exit_json(msg="file already exists", src=src, dest=dest, changed=False)
		if os.access(b_dest, os.R_OK):
			checksum_dest = module.sha1(dest)
	# 目录不存在，退出执行
	elif not os.path.exists(os.path.dirname(b_dest)):
		try:
			os.stat(os.path.dirname(b_dest))
		except OSError:
			e = get_exception()
			if "permission denied" in to_native(e).lower():
				module.fail_json(msg="Destination directory %s is not accessible" % (os.path.dirname(dest)))
		module.fail_json(msg="Destination directory %s does not exist" % (os.path.dirname(dest)))

	# 源文件与目标文件sha1值不一致时覆盖源文件
	if checksum_src != checksum_dest:
	  if not module.check_mode:
		try:
			if remote_src:
				shutil.copy(b_src, b_dest)
			else:
				module.atomic_move(b_src, b_dest)
		except IOError:
			module.fail_json(msg="failed to copy: %s to %s" % (src, dest))
		changed = True

	else:
		changed = False
	
	# 返回值
	res_args = dict(
		dest=dest, src=src, checksum=checksum_src, changed=changed
	)

	module.exit_json(**res_args)


if __name__ == '__main__':
	main()
```

## playbook
---

```
---
- hosts: node1
  gather_facts: false
  
  tasks:
   - name: copy file to remote
	 le_copy: src=/tmp/foo dest=/tmp/test

   - name: unforce copy file to remote
	 le_copy: src=/tmp/foo dest=/tmp/test force=false

   - name: copy file on remote
	 le_copy: src=/tmp/foo dest=/tmp/123 remote_src=true
```

## 执行结果
---


![image.png](/assets/images/Ansible/3629406-8bfc68022b8ceeb2.png)

## github

https://github.com/lework/Ansible-dev
