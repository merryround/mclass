pipeline { 
    agent any // 어떤 에이전트(실행 서버)에서든 실행 가능

    tools { // tools: jenkins tools에 등록된 도구 사용
        maven 'maven 3.9.12' // jenkins Tools에 등록한 이름과 정확히 일치해야 함
    }

    environment {
        // 배포에 필요한 변수 설정
        DOCKER_IMAGE = "demo-app" //도커 이미지 이름
        CONTAINER_NAME = "springboot-container" //도커 컨테이너 이름
        JAR_FILE_NAME = "app.jar" //복사할 jar 파일 이름
        PORT = "8081" //컨테이너와 연결할 포트

        REMOTE_USER = "ec2-user" //원격(spring) 서버 사용자 이름
        REMOTE_HOST = "13.125.18.12" //원격(spring) 서버 public ip

        REMOTE_DIR = "/home/ec2-user/deploy" //원격 서버 배포 경로
        SSH_CREDENTIALS_ID = "eef68453-9c02-4cd2-8494-8264edf20715" //RSA Credentials
    }

    stages { // stages: 실제 자동 빌드를 수행하는 단계 정의
        stage('Git Checkout') { // 수행 단계 구분
            steps { // 실제 수행할 명령어 정의
                // Jenkins가 연결된 Git 저장소에서 최신 코드 체크아웃
                checkout scm
            }            
        }

        stage('Maven Build') {
            steps {
                // 테스트는 건너뛰고 Maven 빌드
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Prepare Jar') {
            steps {
                // 빌드 결과무을 app.jar라는 고정 이름으로 복사
                sh 'cp target/demo-0.0.1-SNAPSHOT.jar ${JAR_FILE_NAME}'
            }
        }

        stage('Copy to Remote Server') {
            steps {
                // 원격 명령 실행 (젠킨스 플러그인 ssh agent 사용)
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    // 배포 디렉토리 생성 (없으면)
                    sh "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} \"mkdir -p ${REMOTE_DIR}\""
                    // JAR와 Dockerfile을 원격 서버로 복사
                    sh "scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${JAR_FILE_NAME} Dockerfile ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"
                }
            }
        }

        stage('Remote Docker Build & Deploy') {
            steps {
                // ENDSSH 뒤에 공백이 없도록 주위
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} << ENDSSH
    cd ${REMOTE_DIR} || exit 1
    docker rm -f ${CONTAINER_NAME} || true
    docker build -t ${DOCKER_IMAGE} .
    docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} ${DOCKER_IMAGE}
ENDSSH
                    """
                }
            }
        }

    }
}