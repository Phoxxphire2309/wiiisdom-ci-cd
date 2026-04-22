pipeline {
  agent any

  environment {
    REPO               = 'Phoxxphire2309/wiiisdom-ci-cd'
    GIT_EXE            = 'C:\\Program Files\\Git\\bin\\git.exe'
    KINESIS_EXE        = 'C:\\wiiisdom\\kinesis'
    WORKSPACE_DIR      = 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Initial Wiiisdom Test Build'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Get Commit SHA') {
      steps {
        script {
          env.COMMIT_SHA = powershell(
            script: '& $env:GIT_EXE rev-parse HEAD',
            returnStdout: true
          ).trim()
          echo "Commit SHA: ${env.COMMIT_SHA}"
        }
      }
    }

    stage('Set Pending Status') {
      steps {
        withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
          powershell '''
            $body = '{"state":"pending","context":"wiiisdom-tests","description":"Wiiisdom tests running..."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/$env:REPO/statuses/$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token $env:GH_TOKEN"; "Content-Type"="application/json" } -Body $body
          '''
        }
      }
    }

    stage('Detect Changed Workbooks') {
      steps {
        script {
          def raw = powershell(
            script: '''
              try {
                $files = & $env:GIT_EXE diff --name-only HEAD~1 HEAD
                $workbooks = $files | Where-Object { $_ -match "\\.twbx?$" }
                if ($workbooks) {
                  $workbooks -join "`n"
                } else {
                  $all = Get-ChildItem -Path "workbooks" -Filter "*.twb*" -ErrorAction SilentlyContinue
                  if ($all) { ($all | ForEach-Object { "workbooks\\" + $_.Name }) -join "`n" } else { "" }
                }
              } catch {
                ""
              }
            ''',
            returnStdout: true
          ).trim()

          env.CHANGED_WORKBOOKS_RAW = raw
          def count = raw ? raw.split('\n').length : 0
          echo "Changed workbooks (${count}): ${raw}"
        }
      }
    }

    stage('Run Wiiisdom Tests') {
      when { expression { env.CHANGED_WORKBOOKS_RAW?.trim() } }
      steps {
        script {
          def workbooks = env.CHANGED_WORKBOOKS_RAW.trim().split('\n')
          def allPassed = true
          def failureDetails = []

          // Create results directory
          powershell 'New-Item -ItemType Directory -Force -Path "results" | Out-Null'

          for (wb in workbooks) {
            wb = wb.trim()
            if (!wb) continue

            def filename = wb.tokenize('/\\').last()
            def name = filename.replaceAll('\\.twbx?$', '')
            def testFile = "wiiisdom\\${name}.json"

            echo "Processing workbook: ${wb}"
            echo "Looking for test file: ${testFile}"

            if (!fileExists(testFile)) {
              echo "No test file found for ${wb} — skipping."
              continue
            }

            echo "Running Wiiisdom tests for ${wb}..."

            def result = powershell(
              script: """
                & \$env:KINESIS_EXE --path \"\$env:WORKSPACE_DIR\\wiiisdom\" --output \"\$env:WORKSPACE_DIR\\results\" --headless \"${name}\"
              """,
              returnStatus: true
            )

            if (result != 0) {
              allPassed = false
              // Try to read failure details from output report
              def reportPath = "results\\${name}.json"
              try {
                def reportText = readFile file: reportPath
                failureDetails << "**${wb}**: ${reportText.take(200)}"
              } catch (e) {
                failureDetails << "**${wb}**: Tests failed — check Jenkins logs for details"
              }
            } else {
              echo "Tests passed for ${wb}"
            }
          }

          env.ALL_PASSED      = allPassed ? 'true' : 'false'
          env.FAILURE_DETAILS = failureDetails.join('\n')
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
        powershell '''
          $body = '{"state":"success","context":"wiiisdom-tests","description":"All Wiiisdom tests passed. Published to Tableau Cloud."}'
          Invoke-RestMethod -Uri "https://api.github.com/repos/$env:REPO/statuses/$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token $env:GH_TOKEN"; "Content-Type"="application/json" } -Body $body
        '''
      }
    }

    failure {
      withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
        script {
          def details = (env.FAILURE_DETAILS ?: 'Pipeline failed — check Jenkins build logs.').replace('"', '\\"')
          powershell """
            \$body = '{"state":"failure","context":"wiiisdom-tests","description":"Wiiisdom tests FAILED. Merge blocked."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/statuses/\$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
          """

          if (env.CHANGE_ID) {
            def comment = "## Wiiisdom Test Results: FAILED\\n\\nThe following workbooks failed:\\n\\n${details}\\n\\n**This PR has NOT been merged.**"
            powershell """
              \$body = '{"body":"${comment}"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:CHANGE_ID/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
            """
          }
        }
      }
    }
  }
}