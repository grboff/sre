import groovy.json.JsonSlurper

@NonCPS
def getSubsystemsList(String text) {
    // Парсим и сразу превращаем в простой список простых мап (ArrayList)
    def json = new JsonSlurper().parseText(text)
    def result = []
    json.subsystems.each { 
        result.add([id: it.id, dc: it.dc, ns: it.namespace, enabled: it.enabled])
    }
    return result
}

pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: helm-tools
    image: dtzar/helm-kubectl:latest
    command: ["cat"]
    tty: true
"""
        }
    }
    
    stages {
        stage('SRE Validation') {
            steps {
                container('helm-tools') {
                    script {
                        echo "🔍 Создание конфигурации..."
                        def jsonConfig = '{"subsystems": [{"id": "mg-branch", "dc": "mg", "namespace": "helloworld-nt-sb-mg", "enabled": true}, {"id": "gm-branch", "dc": "gm", "namespace": "helloworld-nt-sb-gm", "enabled": true}]}'
                        writeFile file: 'subsystems.json', text: jsonConfig
                    }
                }
            }
        }

        stage('Multi-DC Deployment') {
            steps {
                container('helm-tools') {
                    script {
                        def rawText = readFile('subsystems.json')
                        // Получаем чистый список БЕЗ объектов LazyMap
                        def subsystems = getSubsystemsList(rawText)
                        
                        // Используем классический цикл
                        for (int i = 0; i < subsystems.size(); i++) {
                            def item = subsystems[i]
                            
                            if (item.enabled) {
                                echo "🚀 Деплой в ЦОД: ${item.dc.toUpperCase()}"
                                
                                // Выполняем действия
                                sh "kubectl create namespace ${item.ns} --dry-run=client -o yaml | kubectl apply -f -"
                                echo "HELM: upgrade --install hwa-${item.dc} ./charts/unimon-agent-only --namespace ${item.ns} --set datacenter=${item.dc}"
                                
                                sh "echo 'Плечо ${item.dc} готово'"
                            }
                        }
                        // Очищаем переменную, чтобы Jenkins точно ничего не сохранял
                        subsystems = null
                    }
                }
            }
        }
    }
    
    post {
        success { echo "❇️ Поздравляю! Пайплайн прошел успешно." }
        failure { echo "🛑 Ошибка. Проверь логи выше." }
    }
}