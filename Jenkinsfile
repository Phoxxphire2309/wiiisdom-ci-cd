pipeline {
  agent any

  environment {
    REPO          = 'Phoxxphire2309/wiiisdom-ci-cd'
    GIT_EXE       = 'C:\\Program Files\\Git\\bin\\git.exe'
    KINESIS_EXE   = 'C:\\Users\\Administrator\\Downloads\\Wiiisdom-for-Tableau-bundle-2026.2-win32\\Wiiisdom-for-Tableau-bundle-2026.2-win32\\kinesis-cli\\kinesis'
    WORKSPACE_DIR = 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Initial Wiiisdom Test Build'
    BROWSER_PATH  = 'C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe'
    BASE_BRANCH   = 'main'
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

    stage('Create Pull Request') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
            script {
              def branch = env.GIT_BRANCH?.replace('origin/', '') ?: 'development'
              echo "Current branch: ${branch}"

              if (branch != env.BASE_BRANCH) {
                // Check if PR already exists
                def existingPR = powershell(
                  script: """
                    try {
                      \$response = Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls?head=Phoxxphire2309:${branch}&base=\$env:BASE_BRANCH&state=open" -Method GET -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" }
                      if (\$response.Count -gt 0) { \$response[0].number } else { "" }
                    } catch { "" }
                  """,
                  returnStdout: true
                ).trim()

                if (existingPR) {
                  env.PR_NUMBER = existingPR
                  echo "PR already exists: #${env.PR_NUMBER}"
                } else {
                  def prNumber = powershell(
                    script: """
                      \$body = '{"title":"Automated: ${branch}","body":"Automatically created by Jenkins pipeline. Wiiisdom tests running...","head":"${branch}","base":"\$env:BASE_BRANCH"}'
                      \$response = Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
                      \$response.number
                    """,
                    returnStdout: true
                  ).trim()
                  env.PR_NUMBER = prNumber
                  echo "Created PR #${env.PR_NUMBER}"
                }
              } else {
                echo "On base branch — skipping PR creation"
              }
            }
          }
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
                & \$env:KINESIS_EXE --canvas-timeout=60 --path \"\$env:WORKSPACE_DIR\\wiiisdom\\${name}.json\" --output \"\$env:WORKSPACE_DIR\\results\" --context-vars \"\$env:WORKSPACE_DIR\\wiiisdom\\context.json\" --accept-language en-US --max-data-rows 250 --nousagedata --driver-name edgedriver --browser-path \"\$env:BROWSER_PATH\" --headless
              """,
              returnStatus: true
            )

            if (result != 0) {
              allPassed = false
              try {
                def reportText = readFile file: "results\\${name}.json"
                failureDetails << "**${wb}**: ${reportText.take(300)}"
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
        script {
          // Post success status
          powershell '''
            $body = '{"state":"success","context":"wiiisdom-tests","description":"All Wiiisdom tests passed. Published to Tableau Cloud."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/$env:REPO/statuses/$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token $env:GH_TOKEN"; "Content-Type"="application/json" } -Body $body
          '''

          // Merge and close PR if one was created
          if (env.PR_NUMBER) {
            echo "Merging PR #${env.PR_NUMBER}..."
            powershell """
              # Comment on PR
              \$comment = '{"body":"## Wiiisdom Tests: PASSED ✅\\n\\nAll tests passed. Merging automatically."}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:PR_NUMBER/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$comment

              # Merge the PR
              \$merge = '{"commit_title":"Auto-merged by Jenkins after Wiiisdom tests passed","merge_method":"merge"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls/\$env:PR_NUMBER/merge" -Method PUT -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$merge
            """
            echo "PR #${env.PR_NUMBER} merged successfully"
          }
        }
      }
    }

    failure {
      withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
        script {
          def details = (env.FAILURE_DETAILS ?: 'Pipeline failed — check Jenkins build logs.').replace('"', '\\"')

          // Post failure status
          powershell """
            \$body = '{"state":"failure","context":"wiiisdom-tests","description":"Wiiisdom tests FAILED. Merge blocked."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/statuses/\$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
          """

          // Comment on PR and close it if one was created
          if (env.PR_NUMBER) {
            echo "Closing PR #${env.PR_NUMBER} due to test failure..."
            powershell """
              # Add failure comment
              \$comment = '{"body":"## Wiiisdom Tests: FAILED ❌\\n\\nThe following workbooks failed their tests:\\n\\n${details}\\n\\n**This PR has been closed. Please fix the failing tests and push again.**"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:PR_NUMBER/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$comment

              # Close the PR
              \$close = '{"state":"closed"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls/\$env:PR_NUMBER" -Method PATCH -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$close
            """
            echo "PR #${env.PR_NUMBER} closed"
          }
        }
      }
    }
  }
}