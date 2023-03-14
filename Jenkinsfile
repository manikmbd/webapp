pipeline {
  agent any 
  tools {
    maven 'Maven'
  }
  stages {
    stage ('Initialize') {
      steps {
        sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
            ''' 
      }
    }
    
    stage ('Secret Scanning') {
      steps {
        sh 'rm trufflehog || true'
        sh 'docker run gesellix/trufflehog --json https://github.com/manikmbd/webapp.git > trufflehog'
        sh 'cat trufflehog'
      }
    }
    
    stage ('SCA') {
      steps {
         sh 'rm owasp* || true'
         sh 'wget "https://raw.githubusercontent.com/manikmbd/webapp/master/owasp-dependency-check.sh" '
         sh 'chmod +x owasp-dependency-check.sh'
         sh 'bash owasp-dependency-check.sh'
         sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
        
      }
    }
	  
    stage ('SAST - SonarQube') {
      steps {
        withSonarQubeEnv('sonar') {
          sh 'mvn sonar:sonar'
          sh 'cat target/sonar/report-task.txt'
        }
      }
    }	  	  
	  
    stage('Trivy Scan') {
            steps {
                // Install trivy
                sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.18.3'
                sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl'

                // Scan all vuln levels
                sh 'mkdir -p reports'
                sh 'trivy filesystem --ignore-unfixed --vuln-type os,library --format template --template "@html.tpl" -o reports/nodjs-scan.html ./nodejs'
                publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'nodjs-scan.html',
                    reportName: 'Trivy Scan',
                    reportTitles: 'Trivy Scan'
                ]

                // Scan again and fail on CRITICAL vulns
                sh 'trivy filesystem --ignore-unfixed --vuln-type os,library --exit-code 1 --severity CRITICAL ./nodejs'

            }

   
    stage ('Build') {
        steps {
            sh 'mvn clean package'
          }
          }

    stage ('Deploy-To-Tomcat') {
            steps {
           sshagent(['tomcat']) {
                sh 'scp -o StrictHostKeyChecking=no target/*.war ubuntu@3.109.202.96:/var/lib/tomcat9/webapps/webapp.war'
              }      
           }       
    }
    
    stage ('Port Scan') {
		    steps {
			sh 'rm nmap* || true'
			sh 'docker run --rm -v "$(pwd)":/data uzyexe/nmap -sS -sV -oX nmap 3.109.202.96'
			sh 'cat nmap'
		    }
	    }
    
    stage ('Nikto Scan - DAST') {
		    steps {
			sh 'rm nikto-output.xml || true'
			sh 'docker pull secfigo/nikto:latest'
			sh 'docker run --user $(id -u):$(id -g) --rm -v $(pwd):/report -i secfigo/nikto:latest -h 3.109.202.96 -p 8080 -output /report/nikto-output.xml'
			sh 'cat nikto-output.xml'   
		    }
	    }
	  
    stage ('Nuclei Template Scan') {
		    steps {
	        sh 'rm nuclei || true'
		sh 'docker pull projectdiscovery/nuclei'
		sh 'docker run -t projectdiscovery/nuclei -u http://3.109.202.96:8080/webapp/ --json > nuclei'
		sh 'cat nuclei'
	    }
	    }
        }
    }
