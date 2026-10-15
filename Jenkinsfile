
node {
    def dockerExe = 'C:\\Users\\mona shree\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
    def imageName = 'monashree1511/university_lms'
    def kubeCredentialId = 'kubeconfig-file'

    env.IMAGE_TAG = "v${env.BUILD_NUMBER}"

    try {
        stage('1. Checkout GitHub') {
            checkout scm
            bat 'dir'
        }

        stage('2. Build Docker Image') {
            bat """
                "${dockerExe}" build -t ${imageName}:%IMAGE_TAG% .
                if errorlevel 1 exit /b 1
            """
        }

        stage('3. Test Docker Image') {
            bat """
                "${dockerExe}" run -d --name lms-test-%BUILD_NUMBER% -p 18080:80 ${imageName}:%IMAGE_TAG%
                if errorlevel 1 exit /b 1

                "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -NoProfile -NonInteractive -Command "$ok = $false; for ($i=0; $i -lt 10; $i++) { try { $r = Invoke-WebRequest -Uri 'http://localhost:18080' -TimeoutSec 3 -UseBasicParsing; if ($r.StatusCode -eq 200) { $ok = $true; break } } catch { Start-Sleep -Seconds 2 } }; if (-not $ok) { exit 1 }"
                if errorlevel 1 (
                    "${dockerExe}" logs lms-test-%BUILD_NUMBER%
                    "${dockerExe}" rm -f lms-test-%BUILD_NUMBER%
                    exit /b 1
                )

                "${dockerExe}" rm -f lms-test-%BUILD_NUMBER%
            """
        }

        stage('4. Push Versioned Image') {
            withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )
            ]) {
                bat '''
                    "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$env:DOCKER_PASS | & 'C:\\Users\\mona shree\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe' login -u $env:DOCKER_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                    if errorlevel 1 exit /b 1
                '''
                bat """
                    "${dockerExe}" push ${imageName}:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                """
            }
        }

        stage('5. Deploy to Kubernetes') {
            withCredentials([
                file(
                    credentialsId: kubeCredentialId,
                    variable: 'KUBECONFIG_FILE'
                )
            ]) {
                bat """
                    set "KUBECONFIG=%KUBECONFIG_FILE%"
                    kubectl get deployment university-lms
                    if errorlevel 1 exit /b 1

                    kubectl get service university-lms-service
                    if errorlevel 1 exit /b 1

                    kubectl get deployment university-lms -o jsonpath="{.spec.template.spec.containers[0].image}" > previous-image.txt
                    if errorlevel 1 exit /b 1

                    kubectl set image deployment/university-lms *=${imageName}:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1

                    kubectl scale deployment/university-lms --replicas=2
                    if errorlevel 1 exit /b 1
                """
            }
        }

        stage('6. Health Check') {
            withCredentials([
                file(
                    credentialsId: kubeCredentialId,
                    variable: 'KUBECONFIG_FILE'
                )
            ]) {
                int rolloutStatus = 0

                timeout(time: 3, unit: 'MINUTES') {
                    rolloutStatus = bat(
                        returnStatus: true,
                        script: '''
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl rollout status deployment/university-lms --timeout=150s
                        '''
                    )
                }

                if (rolloutStatus != 0) {
                    error('Deployment rollout failed; automatic rollback will start.')
                }

                bat '''
                    set "KUBECONFIG=%KUBECONFIG_FILE%"
                    kubectl get pods
                    kubectl get deployment university-lms
                    kubectl get service university-lms-service
                '''
            }
        }

        currentBuild.result = 'SUCCESS'
    } catch (err) {
        stage('7. Automatic Rollback') {
            withCredentials([
                file(
                    credentialsId: kubeCredentialId,
                    variable: 'KUBECONFIG_FILE'
                )
            ]) {
                bat '''
                    set "KUBECONFIG=%KUBECONFIG_FILE%"
                    kubectl rollout undo deployment/university-lms
                    if errorlevel 1 exit /b 1
                    kubectl rollout status deployment/university-lms --timeout=150s
                    if errorlevel 1 exit /b 1
                    kubectl get pods
                '''
            }

            error("Release failed. Rollback attempted. Original error: ${err}")
        }
    }
}
