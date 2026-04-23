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

          // Skip pipeline if this is a checksums-only commit from Jenkins
          def lastMessage = powershell(
            script: '& $env:GIT_EXE log -1 --pretty=%B',
            returnStdout: true
          ).trim()
          if (lastMessage.contains('[skip ci]')) {
            echo "Skipping pipeline — checksums update commit detected"
            currentBuild.result = 'SUCCESS'
            error('skip')
          }
        }
      }
    }

    stage('Generate and Compare Checksums') {
      steps {
        withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
          script {
            def branch = env.GIT_BRANCH?.replace('origin/', '') ?: 'development'
            env.CURRENT_BRANCH = branch

            def raw = powershell(
              script: '''
                $changedWorkbooks = [System.Collections.Generic.List[string]]::new()

                # Get all current workbooks
                $workbooks = Get-ChildItem -Path "workbooks" -Filter "*.twb*" -ErrorAction SilentlyContinue
                if (-not $workbooks) {
                  Write-Output ""
                  exit 0
                }

                # Load existing checksums if they exist
                $oldMap = @{}
                if (Test-Path "checksums.txt") {
                  Get-Content "checksums.txt" | ForEach-Object {
                    $parts = $_ -split "\s+", 2
                    if ($parts.Count -eq 2) { $oldMap[$parts[1].Trim()] = $parts[0].Trim() }
                  }
                }

                # Compute new checksums and compare
                $newLines = @()
                foreach ($wb in $workbooks) {
                  $hash = (Get-FileHash $wb.FullName -Algorithm SHA256).Hash.ToLower()
                  $path = "workbooks/" + $wb.Name
                  $newLines += "$hash  $path"

                  # Check if changed or new
                  if (-not $oldMap.ContainsKey($path) -or $oldMap[$path] -ne $hash) {
                    $changedWorkbooks.Add($path)
                  }
                }

                # Write updated checksums file
                $newLines | Set-Content "checksums.txt"

                if ($changedWorkbooks.Count -gt 0) { $changedWorkbooks -join "`n" } else { "" }
              ''',
              returnStdout: true
            ).trim()

            env.CHANGED_WORKBOOKS_RAW = raw
            def count = raw ? raw.split('\n').length : 0
            echo "Changed workbooks (${count}): ${raw}"

            // Commit updated checksums back to repo
            powershell """
              & \$env:GIT_EXE config user.email "jenkins@ci"
              & \$env:GIT_EXE config user.name "Jenkins CI"
              & \$env:GIT_EXE add checksums.txt
              \$status = & \$env:GIT_EXE status --porcelain
              if (\$status) {
                & \$env:GIT_EXE commit -m "ci: update checksums [skip ci]"
                & \$env:GIT_EXE push https://\$env:GH_TOKEN@github.com/\$env:REPO.git HEAD:${branch}
              } else {
                Write-Host "No checksum changes to commit"
              }
            """
          }
        }
      }
    }

    stage('Create Pull Request') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
            script {
              def branch = env.CURRENT_BRANCH ?: 'development'

              if (branch != env.BASE_BRANCH) {
                // Check if PR already exists
                def existingPR = powershell(
                  script: """
                    try {
                      \$response = Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls?head=Phoxxphire2309:${branch}&base=${env.BASE_BRANCH}&state=open" -Method GET -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" }
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
                      \$body = '{"title":":rocket: ${branch} -> ${env.BASE_BRANCH}","body":"## :robot: Automated CI/CD Pipeline\\n\\n**Branch:** \`${branch}\`\\n**Base:** \`${env.BASE_BRANCH}\`\\n**Triggered by:** Jenkins push event\\n\\n---\\n\\n### :clipboard: What happens next\\n\\n- Wiiisdom tests will run against all changed workbooks\\n- If tests **pass** this PR will be automatically merged and workbooks published to Tableau Cloud\\n- If tests **fail** this PR will be closed with failure details\\n\\n---\\n*This PR was automatically created by Jenkins CI/CD*","head":"${branch}","base":"${env.BASE_BRANCH}"}'
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

    stage('Run Wiiisdom Tests') {
      when { expression { env.CHANGED_WORKBOOKS_RAW?.trim() } }
      steps {
        script {
          def workbooks = env.CHANGED_WORKBOOKS_RAW.trim().split('\n')
          def allPassed = true
          def failureDetails = []
          def passedList = []

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
                failureDetails << "| \`${wb}\` | :x: Failed | ${reportText.take(150)} |"
              } catch (e) {
                failureDetails << "| \`${wb}\` | :x: Failed | Check Jenkins logs for details |"
              }
            } else {
              passedList << "| \`${wb}\` | :white_check_mark: Passed | Published to Tableau Cloud |"
              echo "Tests passed for ${wb}"
            }
          }

          env.ALL_PASSED      = allPassed ? 'true' : 'false'
          env.FAILURE_DETAILS = failureDetails.join('\n')
          env.PASSED_DETAILS  = passedList.join('\n')
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
            def branch = env.CURRENT_BRANCH ?: 'development'
            def passed = env.PASSED_DETAILS ?: '| No workbooks changed | - | - |'
            echo "Merging PR #${env.PR_NUMBER}..."
            powershell """
              \$comment = '{"body":"## :white_check_mark: Wiiisdom Tests Passed\\n\\nAll tests passed successfully. This PR has been automatically merged and workbooks published to Tableau Cloud.\\n\\n### Results\\n\\n| Workbook | Status | Action |\\n|----------|--------|--------|\\n${passed}\\n\\n---\\n*Automatically merged by Jenkins CI/CD*"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:PR_NUMBER/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$comment

              \$merge = '{"commit_title":":rocket: ${branch} -> ${env.BASE_BRANCH}: Wiiisdom tests passed","commit_message":"All Wiiisdom tests passed. Workbooks published to Tableau Cloud.","merge_method":"squash"}'
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
          def details = env.FAILURE_DETAILS ?: '| Unknown | :x: Failed | Check Jenkins build logs |'
          def branch = env.CURRENT_BRANCH ?: 'development'

          // Post failure status
          powershell """
            \$body = '{"state":"failure","context":"wiiisdom-tests","description":"Wiiisdom tests FAILED. Merge blocked."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/statuses/\$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
          """

          // Comment on PR and close it if one was created
          if (env.PR_NUMBER) {
            echo "Closing PR #${env.PR_NUMBER} due to test failure..."
            powershell """
              \$comment = '{"body":"## :x: Wiiisdom Tests Failed\\n\\nOne or more Wiiisdom tests failed. This PR has been closed automatically.\\n\\n### Failed Workbooks\\n\\n| Workbook | Status | Details |\\n|----------|--------|---------|\\n${details}\\n\\n### :wrench: Next Steps\\n\\n1. Review the failure details above\\n2. Fix the failing tests or workbook\\n3. Push your changes to \`${branch}\` to trigger a new pipeline run\\n\\n---\\n*Automatically closed by Jenkins CI/CD*"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:PR_NUMBER/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$comment

              \$close = '{"state":"closed"}'
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls/\$env:PR_NUMBER" -Method PATCH -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$close
            """
            echo "PR #${env.PR_NUMBER} closed"
          }
        }
      }
    }

    aborted {
      echo "Pipeline skipped — no changes detected or skip ci commit"
    }
  }
}