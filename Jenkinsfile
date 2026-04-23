pipeline {
  agent any

  environment {
    REPO          = 'Phoxxphire2309/wiiisdom-ci-cd'
    GIT_EXE       = 'C:\\Program Files\\Git\\bin\\git.exe'
    KINESIS_EXE   = 'C:\\Users\\Administrator\\Downloads\\Wiiisdom-for-Tableau-bundle-2026.2-win32\\Wiiisdom-for-Tableau-bundle-2026.2-win32\\kinesis-cli\\kinesis'
    WORKSPACE_DIR = 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Initial Wiiisdom Test Build'
    BROWSER_PATH  = 'C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe'
    BASE_BRANCH   = 'main'
    SKIP_CI       = 'false'
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

          def lastMessage = powershell(
            script: '& $env:GIT_EXE log -1 --pretty=%B',
            returnStdout: true
          ).trim()
          if (lastMessage.contains('[skip ci]')) {
            echo "Skipping pipeline — skip ci commit detected"
            env.SKIP_CI = 'true'
          } else {
            env.SKIP_CI = 'false'
          }
        }
      }
    }

    stage('Detect Changed Workbooks') {
      when { expression { env.SKIP_CI != 'true' } }
      steps {
        script {
          def branch = env.GIT_BRANCH?.replace('origin/', '') ?: 'development'
          env.CURRENT_BRANCH = branch

          def raw = powershell(
            script: '''
              try {
                $changedFiles = & $env:GIT_EXE diff --name-only HEAD~1 HEAD 2>$null
                $addedFiles = & $env:GIT_EXE diff --name-only --diff-filter=A HEAD~1 HEAD 2>$null

                $allChanged = @()
                if ($changedFiles) { $allChanged += $changedFiles }
                if ($addedFiles)   { $allChanged += $addedFiles }

                $workbooks = $allChanged | Sort-Object -Unique | Where-Object {
                  $_ -like "workbooks*" -and (
                    $_ -like "*.twb" -or $_ -like "*.twbx"
                  ) -and $_ -like "* v*.*.*"
                }

                if ($workbooks) { $workbooks -join "`n" } else { "" }
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

    stage('Create Pull Request') {
      when { expression { env.SKIP_CI != 'true' && env.CHANGED_WORKBOOKS_RAW?.trim() } }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
            script {
              def branch = env.CURRENT_BRANCH ?: 'development'

              if (branch != env.BASE_BRANCH) {
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
                  def workbookList = env.CHANGED_WORKBOOKS_RAW.trim().split('\n').collect { "| :bar_chart: ${it.trim()} | :hourglass_flowing_sand: Pending |" }.join('\\n')
                  def prTitle = "CI: ${branch} -> ${env.BASE_BRANCH}"
                  def prBody = ":robot: **Automated CI/CD Pipeline**\\n\\n---\\n\\n" +
                    "| Detail | Value |\\n" +
                    "|--------|-------|\\n" +
                    "| :twisted_rightwards_arrows: Branch | ${branch} |\\n" +
                    "| :dart: Target | ${env.BASE_BRANCH} |\\n" +
                    "| :zap: Triggered by | Jenkins push event |\\n\\n" +
                    "---\\n\\n" +
                    "### :bar_chart: Workbooks Queued for Testing\\n\\n" +
                    "| Workbook | Status |\\n" +
                    "|----------|--------|\\n" +
                    "${workbookList}\\n\\n" +
                    "---\\n\\n" +
                    "### :clipboard: Pipeline Steps\\n\\n" +
                    "- :white_check_mark: Pull request created\\n" +
                    "- :hourglass_flowing_sand: Wiiisdom tests running...\\n" +
                    "- :soon: Results will be posted automatically\\n\\n" +
                    "---\\n\\n" +
                    "> :bulb: If tests **pass** this PR will be automatically merged\\n" +
                    "> If tests **fail** this PR will be closed with full failure details\\n\\n" +
                    "---\\n" +
                    "*:robot: This PR was automatically created by Jenkins CI/CD*"

                  writeFile file: 'pr_body.json', text: "{\"title\":\"${prTitle}\",\"body\":\"${prBody}\",\"head\":\"${branch}\",\"base\":\"${env.BASE_BRANCH}\"}"

                  def prNumber = powershell(
                    script: """
                      \$body = Get-Content 'pr_body.json' -Raw
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
      when { expression { env.SKIP_CI != 'true' && env.CHANGED_WORKBOOKS_RAW?.trim() } }
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
      when { expression { env.SKIP_CI != 'true' && env.CHANGED_WORKBOOKS_RAW?.trim() } }
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

            // Strip version suffix e.g. "Session 1 v1.0.1.twb" -> "Session 1"
            def name = filename.replaceAll(' v[0-9]+\\.[0-9]+\\.[0-9]+\\.twbx?$', '')
                               .replaceAll('\\.twbx?$', '')

            def testFile = "wiiisdom\\${name}.json"

            echo "Processing workbook: ${wb}"
            echo "Derived test name: ${name}"
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
                failureDetails << "| :bar_chart: ${wb} | :x: Failed | ${reportText.take(150)} |"
              } catch (e) {
                failureDetails << "| :bar_chart: ${wb} | :x: Failed | Check Jenkins logs for details |"
              }
            } else {
              passedList << "| :bar_chart: ${wb} | :white_check_mark: Passed |"
              echo "Tests passed for ${wb}"
            }
          }

          env.ALL_PASSED      = allPassed ? 'true' : 'false'
          env.FAILURE_DETAILS = failureDetails.join('\\n')
          env.PASSED_DETAILS  = passedList.join('\\n')
        }
      }
    }
  }

  post {
    success {
      script {
        if (env.SKIP_CI == 'true') {
          echo "Skip CI build — no post actions needed"
          return
        }
        if (!env.CHANGED_WORKBOOKS_RAW?.trim()) {
          echo "No workbooks changed — no post actions needed"
          return
        }
        withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
          powershell '''
            $body = '{"state":"success","context":"wiiisdom-tests","description":"All Wiiisdom tests passed."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/$env:REPO/statuses/$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token $env:GH_TOKEN"; "Content-Type"="application/json" } -Body $body
          '''

          if (env.PR_NUMBER) {
            def branch = env.CURRENT_BRANCH ?: 'development'
            def passed = env.PASSED_DETAILS ?: '| :bar_chart: No workbooks changed | - |'
            def successBody = ":white_check_mark: **Wiiisdom Tests Passed**\\n\\n---\\n\\n" +
              "All tests completed successfully. This PR has been automatically merged.\\n\\n" +
              "### :trophy: Results\\n\\n" +
              "| Workbook | Status |\\n" +
              "|----------|--------|\\n" +
              "${passed}\\n\\n" +
              "---\\n\\n" +
              "> :tada: Great work! All workbooks passed their Wiiisdom tests.\\n\\n" +
              "---\\n" +
              "*:robot: Automatically merged by Jenkins CI/CD*"

            def mergeTitle = "CI: ${branch} -> ${env.BASE_BRANCH} - Wiiisdom tests passed"

            writeFile file: 'success_comment.json', text: "{\"body\":\"${successBody}\"}"
            writeFile file: 'merge_body.json', text: "{\"commit_title\":\"${mergeTitle}\",\"commit_message\":\"All Wiiisdom tests passed successfully.\",\"merge_method\":\"merge\"}"

            echo "Merging PR #${env.PR_NUMBER}..."
            powershell """
              \$comment = Get-Content 'success_comment.json' -Raw
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:PR_NUMBER/comments" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$comment

              try {
                Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls/\$env:PR_NUMBER/update-branch" -Method PUT -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body '{}'
                Write-Host "Branch updated — waiting for GitHub to process..."
                Start-Sleep -Seconds 10
              } catch {
                Write-Host "Branch already up to date"
              }

              \$merge = Get-Content 'merge_body.json' -Raw
              Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/pulls/\$env:PR_NUMBER/merge" -Method PUT -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$merge
            """
            echo "PR #${env.PR_NUMBER} merged successfully"
          }
        }
      }
    }

    failure {
      script {
        if (env.SKIP_CI == 'true') {
          echo "Skip CI build — no post actions needed"
          return
        }
        if (!env.CHANGED_WORKBOOKS_RAW?.trim()) {
          echo "No workbooks changed — no post actions needed"
          return
        }
        withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
          def details = env.FAILURE_DETAILS ?: '| :bar_chart: Unknown | :x: Failed | Check Jenkins build logs |'
          def branch = env.CURRENT_BRANCH ?: 'development'

          powershell """
            \$body = '{"state":"failure","context":"wiiisdom-tests","description":"Wiiisdom tests FAILED. Merge blocked."}'
            Invoke-RestMethod -Uri "https://api.github.com/repos/\$env:REPO/statuses/\$env:COMMIT_SHA" -Method POST -Headers @{ Authorization="token \$env:GH_TOKEN"; "Content-Type"="application/json" } -Body \$body
          """

          if (env.PR_NUMBER) {
            def failBody = ":x: **Wiiisdom Tests Failed**\\n\\n---\\n\\n" +
              "One or more Wiiisdom tests failed. This PR has been closed automatically.\\n\\n" +
              "### :mag: Failed Workbooks\\n\\n" +
              "| Workbook | Status | Details |\\n" +
              "|----------|--------|---------|\\n" +
              "${details}\\n\\n" +
              "---\\n\\n" +
              "### :wrench: Next Steps\\n\\n" +
              "1. :eyes: Review the failure details above\\n" +
              "2. :hammer: Fix the failing tests or workbook\\n" +
              "3. :arrow_up: Bump the version and push to ${branch} to trigger a new pipeline run\\n\\n" +
              "> :warning: This PR has been closed. A new PR will be created automatically on your next push.\\n\\n" +
              "---\\n" +
              "*:robot: Automatically closed by Jenkins CI/CD*"

            writeFile file: 'fail_comment.json', text: "{\"body\":\"${failBody}\"}"

            echo "Closing PR #${env.PR_NUMBER} due to test failure..."
            powershell """
              \$comment = Get-Content 'fail_comment.json' -Raw
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
      echo "Pipeline aborted"
    }
  }
}