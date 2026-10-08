// ============================================================
// Pipeline: JSON Update + Git Push + Maven Build + SonarQube + Nexus
// ============================================================

def abortBuild(String msg) {
    currentBuild.result = 'ABORTED'
    error(msg)
}

pipeline {

    agent any

    // ========================================================
    // JENKINS TOOLS
    // ========================================================
    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    // ========================================================
    // PARAMETERS
    // ========================================================
    parameters {

        booleanParam(
            name: 'DEV_JSON',
            defaultValue: false,
            description: 'Update node/dev.json'
        )

        booleanParam(
            name: 'PROD_JSON',
            defaultValue: false,
            description: 'Update node/prod.json'
        )

        booleanParam(
            name: 'STAGE_JSON',
            defaultValue: false,
            description: 'Update node/stage.json'
        )

        booleanParam(
            name: 'UAT_JSON',
            defaultValue: false,
            description: 'Update node/uat.json'
        )

        string(
            name: 'P_ENVIRONMENT',
            defaultValue: '',
            description: 'environment'
        )

        string(
            name: 'P_NODE_NAME',
            defaultValue: '',
            description: 'nodeName'
        )

        string(
            name: 'P_NODE_TYPE',
            defaultValue: '',
            description: 'nodeType'
        )

        string(
            name: 'P_REGION',
            defaultValue: '',
            description: 'region'
        )

        string(
            name: 'P_AZ',
            defaultValue: '',
            description: 'availabilityZone'
        )

        string(
            name: 'P_INSTANCE_TYPE',
            defaultValue: '',
            description: 'instanceType'
        )

        string(
            name: 'P_OS',
            defaultValue: '',
            description: 'os'
        )

        string(
            name: 'P_K8S_ROLE',
            defaultValue: '',
            description: 'kubernetes.role'
        )

        string(
            name: 'P_K8S_VERSION',
            defaultValue: '',
            description: 'kubernetes.version'
        )

        string(
            name: 'P_CPU',
            defaultValue: '',
            description: 'resources.cpu - example: 2'
        )

        string(
            name: 'P_MEMORY',
            defaultValue: '',
            description: 'resources.memory - example: 4Gi'
        )

        string(
            name: 'P_DISK',
            defaultValue: '',
            description: 'resources.disk - example: 50Gi'
        )

        string(
            name: 'P_LABEL_ENV',
            defaultValue: '',
            description: 'labels.environment'
        )

        string(
            name: 'P_LABEL_TEAM',
            defaultValue: '',
            description: 'labels.team'
        )
    }

    // ========================================================
    // ENVIRONMENT
    // ========================================================
    environment {

        REPO = 'github.com/venugh8899/flipkart.git'

        BRANCH = 'main'
    }

    // ========================================================
    // STAGES
    // ========================================================
    stages {

        // ====================================================
        // 1. VALIDATE INPUTS
        // ====================================================
        stage('Validate Inputs') {

            steps {

                script {

                    echo '=========================================='
                    echo 'VALIDATING INPUTS'
                    echo '=========================================='

                    def files = []

                    if (params.DEV_JSON) {
                        files << 'node/dev.json'
                    }

                    if (params.PROD_JSON) {
                        files << 'node/prod.json'
                    }

                    if (params.STAGE_JSON) {
                        files << 'node/stage.json'
                    }

                    if (params.UAT_JSON) {
                        files << 'node/uat.json'
                    }

                    if (files.isEmpty()) {

                        abortBuild(
                            'ABORTED: Select at least one JSON file.'
                        )
                    }

                    env.FILES = files.join(' ')

                    def valueNames = [
                        'P_ENVIRONMENT',
                        'P_NODE_NAME',
                        'P_NODE_TYPE',
                        'P_REGION',
                        'P_AZ',
                        'P_INSTANCE_TYPE',
                        'P_OS',
                        'P_K8S_ROLE',
                        'P_K8S_VERSION',
                        'P_CPU',
                        'P_MEMORY',
                        'P_DISK',
                        'P_LABEL_ENV',
                        'P_LABEL_TEAM'
                    ]

                    def missing = []
                    def entered = []

                    valueNames.each { name ->

                        def value = params[name]

                        if (
                            value == null ||
                            value.toString().trim() == '' ||
                            value.toString().trim() == 'no-change'
                        ) {

                            missing << name

                        } else {

                            entered << "${name}=${value}"
                        }
                    }

                    if (!missing.isEmpty()) {

                        abortBuild(
                            "ABORTED: Missing values: " +
                            "${missing.join(', ')}"
                        )
                    }

                    def cpu = params.P_CPU.toString().trim()

                    if (!cpu.matches('[0-9]+')) {

                        abortBuild(
                            "ABORTED: P_CPU must contain numbers only. " +
                            "Received: ${cpu}"
                        )
                    }

                    currentBuild.displayName =
                        "#${env.BUILD_NUMBER} ${files.join(', ')}"

                    currentBuild.description =
                        "Files: ${files.join(', ')}"

                    echo "Selected Files : ${env.FILES}"

                    echo "Input Values   : ${entered.join(', ')}"

                    echo 'Validation successful.'
                }
            }
        }

        // ====================================================
        // 2. CHECKOUT
        // ====================================================
        stage('Checkout') {

            steps {

                echo '=========================================='
                echo 'CHECKOUT SOURCE CODE'
                echo '=========================================='

                cleanWs()

                git(
                    branch: "${env.BRANCH}",
                    url: "https://${env.REPO}",
                    credentialsId: 'github-creds'
                )
            }
        }

        // ====================================================
        // 3. UPDATE JSON
        // ====================================================
        stage('Update JSON') {

            steps {

                sh '''
                    set -e

                    echo "Creating jq filter..."

                    cat > filter.jq <<'JQ'

def p($k):
    ($ENV[$k] // "");

def setstr($path; $k):
    if p($k) != "" then
        setpath($path; p($k))
    else
        .
    end;

setstr(["environment"]; "P_ENVIRONMENT")
| setstr(["nodeName"]; "P_NODE_NAME")
| setstr(["nodeType"]; "P_NODE_TYPE")
| setstr(["region"]; "P_REGION")
| setstr(["availabilityZone"]; "P_AZ")
| setstr(["instanceType"]; "P_INSTANCE_TYPE")
| setstr(["os"]; "P_OS")
| setstr(["kubernetes","role"]; "P_K8S_ROLE")
| setstr(["kubernetes","version"]; "P_K8S_VERSION")
| setstr(["resources","cpu"]; "P_CPU")
| setstr(["resources","memory"]; "P_MEMORY")
| setstr(["resources","disk"]; "P_DISK")
| setstr(["labels","environment"]; "P_LABEL_ENV")
| setstr(["labels","team"]; "P_LABEL_TEAM")

JQ

                    for f in $FILES
                    do

                        echo "Processing: $f"

                        if [ ! -f "$f" ]; then

                            echo "ERROR: File not found: $f"

                            exit 1
                        fi

                        jq -f filter.jq "$f" > tmp.json

                        mv tmp.json "$f"

                        echo "=========================================="
                        echo "UPDATED FILE: $f"
                        echo "=========================================="

                        cat "$f"

                    done

                    rm -f filter.jq
                '''
            }
        }

        // ====================================================
        // 4. REVIEW CHANGES
        // ====================================================
        stage('Review Changes') {

            steps {

                script {

                    def rc = sh(
                        returnStatus: true,
                        script: 'git diff --quiet'
                    )

                    if (rc != 0) {

                        env.HAS_CHANGES = 'true'

                    } else {

                        env.HAS_CHANGES = 'false'
                    }
                }

                sh '''
                    echo "=========================================="
                    echo "GIT DIFF SUMMARY"
                    echo "=========================================="

                    git --no-pager diff --stat

                    echo "=========================================="
                    echo "GIT DIFF DETAILS"
                    echo "=========================================="

                    git --no-pager diff
                '''

                script {

                    if (env.HAS_CHANGES == 'false') {

                        echo 'No changes detected.'
                    }
                }
            }
        }

        // ====================================================
        // 5. COMMIT AND PUSH
        // ====================================================
        stage('Commit & Push') {

            when {

                expression {

                    env.HAS_CHANGES == 'true'
                }
            }

            steps {

                withCredentials(
                    [
                        usernamePassword(
                            credentialsId: 'github-creds',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]
                ) {

                    sh '''
                        set -e

                        echo "Configuring Git..."

                        git config user.name "Jenkins"

                        git config user.email "jenkins@example.com"

                        echo "Adding files..."

                        git add $FILES

                        if git diff --cached --quiet
                        then

                            echo "No staged changes."

                        else

                            echo "Creating commit..."

                            git commit \
                                -m "[skip ci] Jenkins: updated $FILES (build #${BUILD_NUMBER})"

                            echo "Pulling latest changes..."

                            git pull \
                                --rebase \
                                "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" \
                                "$BRANCH"

                            echo "Pushing changes..."

                            git push \
                                "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" \
                                "HEAD:${BRANCH}"

                            echo "Git push successful."

                        fi
                    '''
                }
            }
        }

        // ====================================================
        // 6. MAVEN BUILD
        // ====================================================
        stage('Maven Build') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "JAVA VERSION"
                    echo "=========================================="

                    java -version

                    echo "=========================================="
                    echo "MAVEN VERSION"
                    echo "=========================================="

                    mvn -version

                    echo "=========================================="
                    echo "CHECKING POM.XML"
                    echo "=========================================="

                    if [ ! -f pom.xml ]; then

                        echo "ERROR: pom.xml not found."

                        echo "Current directory:"
                        pwd

                        echo "Files:"
                        ls -la

                        exit 1
                    fi

                    echo "pom.xml found."

                    echo "=========================================="
                    echo "STARTING MAVEN BUILD"
                    echo "=========================================="

                    mvn -B clean package

                    echo "=========================================="
                    echo "MAVEN BUILD SUCCESS"
                    echo "=========================================="
                '''
            }
        }

        // ====================================================
        // 7. SONARQUBE ANALYSIS
        // ====================================================
        stage('SonarQube Analysis') {

            steps {

                echo '=========================================='
                echo 'SONARQUBE ANALYSIS'
                echo '=========================================='

                withSonarQubeEnv('sonarqube') {

                    sh '''
                        set -e

                        echo "Starting SonarQube analysis..."

                        mvn -B \
                            org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=flipkart \
                            -Dsonar.projectName=flipkart

                        echo "=========================================="
                        echo "SONARQUBE ANALYSIS SUCCESS"
                        echo "=========================================="
                    '''
                }
            }
        }

        // ====================================================
        // 8. NEXUS UPLOAD
        // ====================================================
        stage('Nexus Upload') {

            steps {

                echo '=========================================='
                echo 'NEXUS ARTIFACT UPLOAD'
                echo '=========================================='

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Creating Maven settings.xml for Nexus..."

                        cat > settings.xml <<EOF
<settings>
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>${NEXUS_USER}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>
</settings>
EOF

                        echo "Starting Maven deploy to Nexus..."

                        mvn -B deploy -s settings.xml

                        rm -f settings.xml

                        echo "=========================================="
                        echo "NEXUS UPLOAD SUCCESS"
                        echo "=========================================="
                        echo "Repository: maven-releases1"
                        echo "URL: http://13.201.93.203:8081/repository/maven-releases1/"
                    '''
                }
            }
        }

        // ====================================================
        // 9. CHECK ARTIFACT
        // ====================================================
        stage('Check Artifact') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "TARGET DIRECTORY"
                    echo "=========================================="

                    ls -lah target/

                    echo "=========================================="
                    echo "GENERATED JAR/WAR"
                    echo "=========================================="

                    find target \
                        -maxdepth 1 \
                        -type f \
                        \\( -name "*.jar" -o -name "*.war" \\) \
                        -print
                '''
            }
        }

        // ====================================================
        // 10. ARCHIVE ARTIFACT
        // ====================================================
        stage('Archive Artifact') {

            steps {

                archiveArtifacts(
                    artifacts: 'target/*.jar, target/*.war',
                    fingerprint: true,
                    allowEmptyArchive: false
                )
            }
        }
    }

    // ========================================================
    // POST ACTIONS
    // ========================================================
    post {

        success {

            echo '=========================================='
            echo 'PIPELINE SUCCESS'
            echo '=========================================='

            echo "Files          : ${env.FILES}"

            echo 'JSON Update    : SUCCESS'

            echo 'Git Push       : SUCCESS'

            echo 'Maven Build    : SUCCESS'

            echo 'SonarQube      : SUCCESS'

            echo 'Nexus Upload   : SUCCESS'

            echo 'Artifact       : ARCHIVED'

            echo "Repository     : https://${env.REPO.replace('.git', '')}"

            echo "Branch         : ${env.BRANCH}"

            echo '=========================================='
        }

        failure {

            echo '=========================================='
            echo 'PIPELINE FAILED'
            echo '=========================================='

            echo 'Check the console output.'

            echo 'Possible issues:'

            echo '1. Input validation'

            echo '2. jq / JSON update'

            echo '3. Git checkout'

            echo '4. GitHub credentials'

            echo '5. Git push'

            echo '6. JDK configuration'

            echo '7. Maven configuration'

            echo '8. pom.xml / Maven build'

            echo '9. SonarQube configuration'

            echo '10. SonarQube analysis'

            echo '11. Nexus credentials / repository / deploy'

            echo '12. Artifact generation'

            echo '=========================================='
        }

        aborted {

            echo '=========================================='
            echo 'PIPELINE ABORTED'
            echo '=========================================='

            echo 'Required input was missing or build was cancelled.'

            echo '=========================================='
        }
    }
}
