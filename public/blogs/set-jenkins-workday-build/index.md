
## 问题背景
  业务GPU服务随着时间的推移，占用的显存会越来越大，与业务沟通，在服务侧没有比较好的解决方案，想针对某些服务进行定时重启。

## 环境信息

| 组件名称 | 组件版本 | 组建功能 |
| --- | --- | --- |
| Centos  | 7.9 | Linux操作系统 |
| Jenkins | Version 2.401.3 | 发版的CI/CD工具 |


## 配置实现

### 日程表配置

业务要求的是工作日，配置的是每周一的11点左右（Jenkins为了避开高负载，实际执行时间会有小波动）
```
H 11 * * 1
```

### pipeline代码

```
pipeline {
    agent any
    options {
        ansiColor('xterm')
    }
    stages {
        stage('节假日检查（仅自动触发）') {
            steps {
                script {
                    // 判断是否为自动触发
                    def isAutoTrigger = isAutoBuild()
                    
                    if (isAutoTrigger) {
                        echo "自动触发模式，检查是否为工作日"
                        if (!isWorkday()) {
                            error("非工作日或节假日，自动构建已跳过")
                        }
                        echo "工作日检查通过，继续执行"
                    } else {
                        echo "手动触发模式，跳过节假日检查"
                    }
                }
            }
        }
        stage('deploy .....') {
         steps {
           sh '''脚本内容'''
         }
        } 
    }
    post {
        success {
            // This block runs if the pipeline succeeds
            echo 'Pipeline succeeded!'
            script {
              FeiShu()
            }
        }
        failure {
            // This block runs if the pipeline fails
            echo 'Pipeline failed!'
            script {
              FeiShu()
            }
        }
    }
}

//飞书
def FeiShu(){
    //过滤掉数值为false的参数，以及release-起始的参数
    def filteredParams = params.findAll { 
                 key, value -> value != false  && !(value.toString().startsWith("release-"))
        }
    sh """
        curl --location --request POST 'https://open.feishu.cn/open-apis/bot/v2/hook/xxxxx' \
        --header 'Content-Type: application/json' \
        --data '{
            "msg_type": "interactive",
            "card": {
                "config": {
                        "wide_screen_mode": true,
                        "enable_forward": true
                },
                "elements": [{
                        "tag": "div",
                        "text": {
                                "content": "$JOB_NAME 作业构建信息: \\n构建ID: $BUILD_ID \\n构建变量: $filteredParams \\n作业状态: $currentBuild.currentResult\\n运行时长: $currentBuild.durationString\\n作业变量详细信息: ${BUILD_URL}parameters/\\n作业日志详细信息: ${BUILD_URL}console\\n",
                                "tag": "lark_md"
                        }
                }, {
                        "actions": [{
                                "tag": "button",
                                "text": {
                                        "content": "作业链接",
                                        "tag": "lark_md"
                                },
                                "url": "$JOB_URL",
                                "type": "default",
                                "value": {}
                        }],
                        "tag": "action"
                }],
                "header": {
                        "title": {
                                "content": "DEVOPS作业构建信息",
                                "tag": "plain_text"
                        }
                }
            }
        } '
    """
}
/**
是否自动构建
**/
def isAutoBuild() {
    def buildCauses = currentBuild.getBuildCauses()
    for (cause in buildCauses) {
        if (cause._class == 'hudson.triggers.TimerTrigger$TimerTriggerCause') {
            return true
        }
    }
    return false
}
/**
 * 判断是否为工作日（跳过周末和法定节假日）
 */
def isWorkday() {
    def today = new Date()
    def dayOfWeek = today[Calendar.DAY_OF_WEEK]
    
    // 跳过周末
    if (dayOfWeek == 1 || dayOfWeek == 7) {
        echo "周末，跳过"
        return false
    }
    
    // 跳过法定节假日
    def holidays = [
        '2026-01-01',                                    // 元旦
        '2026-02-16', '2026-02-17', '2026-02-18',       // 春节
        '2026-02-19', '2026-02-20',
        '2026-04-06',                                    // 清明
        '2026-05-01',                                    // 劳动节
        '2026-06-22',                                    // 端午
        '2026-09-21',                                    // 中秋
        '2026-10-01', '2026-10-02', '2026-10-03',       // 国庆
        '2026-10-04', '2026-10-05', '2026-10-06', '2026-10-07'
    ]
    
    def dateStr = today.format('yyyy-MM-dd')
    if (dateStr in holidays) {
        echo "法定节假日 $dateStr，跳过"
        return false
    }
    
    echo "工作日，继续执行"
    return true
}

```



## 成果截图

## 总结