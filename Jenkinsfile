pipeline {
  agent any

  environment {
    WIIISDOM_LICENSE   = credentials('WIIISDOM_LICENSE_KEY')
    TC_TOKEN_NAME      = credentials('TABLEAU_CLOUD_TOKEN_NAME')
    TC_TOKEN_SECRET    = credentials('TABLEAU_CLOUD_TOKEN_SECRET')
    TC_SITE_ID         = credentials('TABLEAU_SITE_ID')
    TC_SERVER_URL      = credentials('TABLEAU_SERVER_URL')
    REPO               = 'Phoxxphire2309/wiiisdom-ci-cd'
    GIT_EXE            = 'C:\\Program Files\\Git\\bin\\git.exe'
    KINESIS_EXE        = 'C:\\wiiisdom\\kinesis'
    LICENSE_FILE       = 'C:\\wiiisdom\\license.key'
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
          env.CHANGED_WORKBOOKS = powershell(
            script: '''
              try {
                $files = & $env:GIT_EXE diff --name-only HEAD~1 HEAD
                $workbooks = $files | Where-Object { $_ -match "\\.twbx?$" }
                if ($workbooks) { $workbooks -join " " } else { "" }
              } catch {
                ""
              }
            ''',
            returnStdout: true
          ).trim()
          echo "Changed workbooks: ${env.CHANGED_WORKBOOKS}"
        }
      }
    }

    stage('Run Wiiisdom Tests') {
      when { expression { env.CHANGED_WORKBOOKS?.trim() } }
      steps {
        script {
          def workbooks = env.CHANGED_WORKBOOKS.trim().split(' ')
          def allPassed = true
          def failureDetails = []

          for (wb in workbooks) {
            def name = wb.tokenize('/').last().replaceAll('\\.twbx?', '')
            def testFile = "wiiisdom\\${name}.json"

            if (!fileExists(testFile)) {
              echo "No test file found for ${wb} — skipping."
              continue
            }

            echo "Running Wiiisdom tests for ${wb}..."

            def result = powershell(
              script: "& \$env:KINESIS_EXE run --workbook \"${wb}\" --test-file \"${testFile}\" --output-format json --output-file \"results_${name}.json\" --license-file \$env:LICENSE_FILE",
              returnStatus: true
            )

            if (result != 0) {
              allPassed = false
              try {
                def report = readJSON file: "results_${name}.json"
                def msg = report?.summary?.failureMessage ?: 'Check results file for details'
                failureDetails << "**${wb}**: ${msg}"
              } catch (e) {
                failureDetails << "**${wb}**: Tests failed (could not parse report)"
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

    stage('Publish to Tableau Cloud') {
      when { expression { env.ALL_PASSED == 'true' } }
      steps {
        powershell '''
          $workbooks = $env:CHANGED_WORKBOOKS -split " "
          foreach ($wb in $workbooks) {
            Write-Host "Publishing $wb to Tableau Cloud..."
            tabcmd publish "$wb" --server "$env:TC_SERVER_URL" --site "$env:TC_SITE_ID" --token-name "$env:TC_TOKEN_NAME" --token-value "$env:TC_TOKEN_SECRET" --overwrite
          }
        '''
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
