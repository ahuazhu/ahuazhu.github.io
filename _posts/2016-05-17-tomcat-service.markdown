---
layout: post
title:  "tomcat service"
date:   2016-05-17 09:00:00 +0800
categories: 基础设施搭建
---

/etc/init.d/tomcat

{% highlight python %}

#!/bin/env python
# coding: utf-8

# Author : xk
# Startup script for tomcat
# date: 2015/08/20
# 加入输出json支持
# 1001: 启动超时
# 1002: 没有war包
# 1003: 没有安装tomcat


# chkconfig: - 80 90

import sys
import os
import time
import re
import commands
import urllib2
import urllib
import json
import linecache

g = {
    'tomcat_pid': 0,
    'tomcat_user': 'nobody',
    'tomcat_start_timeout': 180,
    'tomcat_stop_timeout': 20,
    'tomcat_home': '/usr/local/tomcat',
    'tomcat_start': '/usr/local/tomcat/bin/startup.sh',
    'tomcat_stop': '/usr/local/tomcat/bin/shutdown.sh',
    'tomcat_log': '/data/applogs/tomcat/catalina.out',
    'tomcat_temp': '/usr/local/tomcat/temp',
    'tomcat_work': '/usr/local/tomcat/work',
    'tomcat_root_xml': '/usr/local/tomcat/conf/Catalina/localhost/ROOT.xml',

    'log_file': '/data/applogs/scripts/log.txt',
    'gc_file': '/data/applogs/heap_trace.txt',
    'war_path': '/data/webapps/{0}/current/{0}.war',
    'title_length': 70,
    'task_length': 50,
    'time': str(time.strftime('%Y-%m-%d %T')),
}


def init(argv):
    process = exec_command(command='ps aux | grep "python /etc/init.d/tomcat" | grep -v grep | wc -l',
                           result=True, enter=False)
    if process == '2':
        echo(u'Can only run one instance', exit_code=1, exit=True)
    g['script_pid'] = os.getpid()
    g['env'] = exec_command("grep 'deployenv' /data/webapps/appenv | cut -d'=' -f 2", result=True).strip()
    g['host_ip'] = exec_command("/sbin/ifconfig |grep 'net addr:' |head -n1 | awk  -F'[: ]' '{print $13}'", result=True)
    g['host_name'] = exec_command('hostname', result=True)
    g['tomcat_pid'] = exec_command('tomcat_pid', result=True)

    if os.path.exists('/data/webapps/paas'):
        g['type'] = 'docker'
        g['war_path'] = '/data/webapps/paas/ROOT'
        project_info = json.loads(http_rpc('http://api.cmdb.dp/api/v0.1/ip/{0}/projects'.format(g['host_ip'])))
        if not project_info:
            echo('not cmdb', exit_code=1, exit=True)
        else:
            g['service_name'] = project_info.get('projects')[0].get('project_name')
    else:
        g['service_name'] = exec_command("hostname | awk -F'[0-9]|-ppe|-sl' '{print $1}'", result=True)
        g['war_path'] = g['war_path'].format(g.get('service_name'))

    if not os.path.exists(os.path.dirname(g['log_file'])):
        os.makedirs(os.path.dirname(g['log_file']))

    if len(argv) == 2:
        if argv[1] == 'json':
            g['json'] = True
            g['result'] = dict(code=0, action=g['action'])
        elif argv[1]:
            g['colour'] = True


def get_colour(msg, colour):
    if g.get('colour'):
        return msg
    if colour == 'red':
        return u'\033[31m{0}\033[0m'.format(msg)
    if colour == 'green':
        return u'\033[32m{0}\033[0m'.format(msg)
    if colour == 'yellow':
        return u'\033[33m{0}\033[0m'.format(msg)


def echo(msg, **kwargs):
    if g.get('json'):
        if kwargs.get('exit'):
            g['result']['code'] = int(kwargs.get('exit_code', 0))
            echo_json()
        return True

    if kwargs.get('length_title'):
        msg = u'{0} {1}'.format(msg, '=' * (g['title_length'] - len(msg)))
    elif kwargs.get('length_task'):
        msg = u'{0} {1}'.format(msg, '.' * (g['task_length'] - len(msg)))
    if kwargs.get('colour'):
        msg = get_colour(msg, kwargs.get('colour'))

    echo_type = kwargs.get('type')
    if echo_type == 'n':
        print msg,
    elif echo_type == 'r':
        print '\r' + msg,
    else:
        print msg

    if kwargs.get('exit'):
        sys.exit(int(kwargs.get('exit_code', 0)))
    else:
        sys.stdout.flush()


def echo_json():
    print json.dumps(g['result'], indent=4)
    sys.exit(0)


def exec_command(command, enter=True, **kwargs):
    if command == 'tomcat_pid':
        command = """ps -ef | grep -v grep | grep 'catalina.startup.Bootstrap start' | awk '{print $2}'"""
        g['command_status'], g['command_result'] = commands.getstatusoutput(command)
        g['tomcat_pid'] = g['command_result']
    else:
        g['command_status'], g['command_result'] = commands.getstatusoutput(command)

    if enter:
        if g['command_status']:
            echo('{0}: {1}'.format(command, g['command_result']), colour='red')
            sys.exit(1)

    if kwargs.get('result'):
        if not g['command_status']:
            return g['command_result']
    elif not g['command_status']:
        return True


def http_rpc(url, method='get', **kwargs):
    if method == 'post':
        headers = kwargs.get('headers', {})
        if kwargs.get('data'):
            headers.update({'Content-Type': 'content-type: text/plain'})
            data = json.dumps(kwargs.get('data'))
        else:
            data = urllib.urlencode(kwargs.get('values', {}))
        request = urllib2.Request(url, headers=headers, data=data)
    else:
        request = urllib2.Request(url, headers=kwargs.get('headers', {}))

    try:
        return urllib2.urlopen(request, timeout=int(kwargs.get('timeout', 10))).read().strip()
    except urllib2.HTTPError, e:
        g['msg'] = '[{0}]: {1}'.format(url, e.code)
    except Exception, e:
        g['msg'] = '[{0}]: {1}'.format(url, str(e))


def start():
    """
    启动tomcat
    """
    echo(u'Start tomcat', length_title=True, colour='yellow')

    # 检测pid
    echo(u'check pid', length_task=True, type='n', colour='green')
    if exec_command('tomcat_pid', result=True):
        echo(u'failed (is running pid: {0})'.format(g['tomcat_pid']), colour='red', exit_code=1, exit=True)
    else:
        echo(u'[OK]', colour='green')

    # 检测tomcat文件夹
    echo(u'check tomcat', length_task=True, type='n', colour='green')
    if not os.path.exists(g.get('tomcat_home')):
        echo(u'failed (not install tomcat)', colour='red', exit_code=1003, exit=True)
    else:
        echo(u'[OK]', colour='green')

    # 检测war包
    echo(u'check war', length_task=True, type='n', colour='green')
    if not os.path.exists(g.get('war_path').format(g.get('service_name'))):
        if g.get('json') or g.get('color'):
            echo(u'Error (not found war)', colour='red', exit=True, exit_code=1002)
        else:
            echo(u'warning (not found war)', colour='red')
    else:
        echo(u'[OK]', colour='green')

    # 初始化日志文件
    echo(u'check log', length_task=True, type='n', colour='green')
    if not os.path.isfile(g.get('tomcat_log')):
        exec_command('touch %s' % g.get('tomcat_log'))
    exec_command('chown %s:%s %s' % (g.get('tomcat_user'), g.get('tomcat_user'), g.get('tomcat_log')))
    echo(u'[OK]', colour='green')

    # 初始化tomcat缓存文
    echo(u'check temp', length_task=True, type='n', colour='green')
    exec_command('rm -rf {0} && mkdir -p {0} && chown {1}:{1} {0}'.format(g['tomcat_temp'], g['tomcat_user']))
    exec_command('rm -rf {0} && mkdir -p {0} && chown {1}:{1} {0}'.format(g['tomcat_work'], g['tomcat_user']))
    echo(u'[OK]', colour='green')

    # 启动tomcat
    echo(u'start ', type='n', colour='green')
    exec_command('/bin/su - {0} -s /bin/bash -c "{1}"'.format(g['tomcat_user'], g['tomcat_start']))

    # 监测关键字
    old_time = echo_time = int(time.time())
    log_tell = os.path.getsize(g.get('tomcat_log'))
    with open(g['tomcat_log'], 'r') as fp:
        fp.seek(log_tell)
        while True:
            line = fp.readline()
            if (echo_time + 2) < time.time():
                echo(u'.', type='n', colour='green')
                echo_time = time.time()
            if old_time + g['tomcat_start_timeout'] < int(time.time()):
                echo(u'failed (start timeout)', exit_code=1001, exit=True, colour='red')
            if not line:
                time.sleep(0.1)
                continue
            start_time = re.search(r'Server startup in (\d+) ms', line)
            if start_time:
                echo(u'[OK] ({0})'.format(start_time.group(0)), colour='green')
                return True


def stop():
    """
    停止tomcat
    使用tomcat的shudown.sh命令停止
    等待20秒，还存在进程，则直接kill掉tomcat进程
    """
    echo(u'Stop tomcat', length_title=True, colour='yellow')

    # 检测pid
    echo(u'check pid', length_task=True, type='n', colour='green')
    if not exec_command('tomcat_pid', result=True):
        echo(u'failed (tomcat not running)', colour='red')
        return False
    else:
        echo(u'[OK]', colour='green')

    # 停止tomcat
    echo(u'Shutdown tomcat ', type='n', colour='green')
    exec_command('/bin/su - %s -s /bin/bash -c "%s"' % (g.get('tomcat_user'), g.get('tomcat_stop')))

    # 检测关键字
    if not os.path.isfile(g.get('tomcat_log')):
        time.sleep(1)
        exec_command('kill -9 %s' % g.get('tomcat_pid'))
        echo(u'[OK] (kill: {0})'.format(g['tomcat_pid']), colour='green')
    else:
        old_time = echo_time = int(time.time())
        while True:
            if (echo_time + 2) < time.time():
                echo(u'.', type='n', colour='green')
                echo_time = time.time()
            if old_time + g['tomcat_stop_timeout'] < int(time.time()):
                exec_command('kill -9 %s' % g.get('tomcat_pid'), enter=False)
                echo(u'[OK] (kill: {0})'.format(g['tomcat_pid']), colour='green')
                break
            if not exec_command('tomcat_pid'):
                echo(u'[OK]', colour='green')
                break
        return True


def status():
    if exec_command('tomcat_pid', result=True):
        echo(u'The process of tomcat (pid={0}) is running...'.format(g['tomcat_pid']), exit=True, colour='green')
    else:
        echo(u'tomcat not running...', colour='red')


def build_time():
    if not g['war_path']:
        echo(u'没有找到war包: {0}'.format(g['war_path']), exit=True, exit_code=1)

    build_file = exec_command('find {0} -name "pom.properties"'.format(g['war_path']), result=True)
    if not build_file:
        echo(u'没有找到pom文件', colour='red', exit=True, exit_code=1)
    else:
        echo(u'build_time: {0}'.format(linecache.getline(build_file, 2).strip()[1:]), colour='green', exit=True)


def gc_file_backup():
    if os.path.isfile(g['gc_file']):
        os.rename(g['gc_file'], g['gc_file'] + '.bak')
        return True


def log():
    for i in linecache.getlines(g['log_file'])[-30:]:
        echo(i, type='n')


def use_help():
    return {
        'command': '/etc/init.d/tomcat ${action}',
        'action': {
            'start': '启动tomcat',
            'stop': '停止tomcat',
            'restart': '重启tomcat',
            'status': '查看tomcat状态',
            'build': '查看war包的build时间',
            'config': '查看脚本配置',
            'log': '查看操作日志'
        }
    }


if __name__ == '__main__':
    if len(sys.argv) not in [2, 3]:
        echo(json.dumps(use_help(), indent=4, ensure_ascii=False, sort_keys=True))
        sys.exit(1)
    else:
        g['action'] = sys.argv[1]
        init(sys.argv[1:])
        if g['action'] not in ['log', 'status']:
            open(g['log_file'], 'a+').write(g['time'] + ' /etc/init.d/tomcat ' + ' '.join(sys.argv[1:]) + '\n')

    try:
        if g['action'] == 'restart':
            stop()
            echo(u'')
            start()

        elif g['action'] == 'start':
            start()

        elif g['action'] == 'stop':
            stop()

        elif g['action'] == 'status':
            status()

        elif g['action'] == 'forcerestart':
            stop()
            start()

        elif g['action'] == 'forcestart':
            start()

        elif g['action'] == 'forcestop':
            stop()

        elif g['action'] == 'build':
            build_time()

        elif g['action'] == 'log':
            log()

        elif g['action'] == 'config':
            echo(json.dumps(g, indent=4, ensure_ascii=False, sort_keys=True))

        else:
            echo(json.dumps(use_help(), indent=4, ensure_ascii=False, sort_keys=True), exit=True, exit_code=1)

        if g.get('json'):
            echo_json()
    except KeyboardInterrupt:
        pass
{% endhighlight %}