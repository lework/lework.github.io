---
layout: post
title: "Ansible Role 系统环境 之【nodejs】"
date: "2017-03-04 15:58:22"
categories: Ansible
excerpt: "Ansible Role: nodejs 安装nodejs 要求 此角色仅在RHEL及其衍生产品上运行。 测试环境 ansible 2.2.1...."
auth: lework
---
* content
{:toc}
{% raw %}

# Ansible Role: nodejs

安装nodejs

## 要求

此角色仅在RHEL及其衍生产品上运行。

## 测试环境

ansible `2.2.1.0`
os `Centos 6.7 X64`

## 角色变量
	software_files_path: "/opt/software"
	software_install_path: "/usr/local"

	nodejs_version: "0.10.36"

	nodejs_file: "node-v{{ nodejs_version }}-linux-x64.tar.gz"
	nodejs_file_path: "{{ software_files_path }}/{{ nodejs_file }}"
	nodejs_file_url: "http://nodejs.org/dist/v{{ nodejs_version }}/{{ nodejs_file }}"

	npm_config_prefix: "{{ nodejs_install_path }}/nodejs"
	npm_registry: "http://npmreg.mirrors.ustc.edu.cn/"


	nodejs_npm_global_packages: 
	 - {'name': 'forever'}


## 依赖

没有

## github地址
https://github.com/lework/Ansible-roles/tree/master/nodejs

## Example Playbook

    - hosts: servers
      roles:
        - nodejs
		
	- hosts: server
      roles:
        - { role: nodejs, nodejs_version: "0.12.9"}
{% endraw %}
