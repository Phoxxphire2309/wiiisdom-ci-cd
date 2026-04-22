// Jenkinsfile — Wiiisdom + Tableau Cloud CI/CD Pipeline (Windows Agent)
pipeline {
  agent any

  environment {
    GITHUB_TOKEN       = credentials('7de4d72f-958e-41ad-b987-ec15556bfcd2')
    WIIISDOM_LICENSE   = credentials('WIIISDOM_LICENSE_KEY')
    TC_TOKEN_NAME      = credentials('TABLEAU_CLOUD_TOKEN_NAME')
    TC_TOKEN_SECRET    = credentials('TABLEAU_CLOUD_TOKEN_SECRET')
    TC_SITE_ID         = credentials('TABLEAU_SITE_ID')
    TC_SERVER_URL      = credentials('TABLEAU_SERVER_URL')
    REPO               = 'Phoxxphire2309/wiiisdom-ci-cd'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Pending Status') {
      steps {
        powershell '''
          $sha = (git rev-parse HEAD).Trim()
          $body = @{ state="pending"; context="wiiisdom-tests"; description="Wiiisdom tests running..." } | ConvertTo-Json
          Invoke-RestMethod `
            -Uri "https://api.github.com/repos/$env:REPO/statuses/$sha" `
            -Method POST `
            -Headers @{ Authorization="token $env:GITHUB_TOKEN"; "Content-Type"="application/json" } `
            -Body $body
        '''
      }
    }

    stage('Detect Changed Workbooks') {
      steps {
        script {
          env.CHANGED_WORKBOOKS = powershell(
            script: '''
              $files = git diff --name-only HEAD~1 HEAD
              $workbooks = $files | Where-Object { $_ -match "\\.twbx?$" }
              $workbooks -join " "
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

            def result = powershell(
              script: """
                wiiisdom-ops run `
                  --workbook "${wb}" `
                  --test-file "${testFile}" `
                  --output-format json `
                  --output-file "results_${name}.json" `
                  --license \$env:WIIISDOM_LICENSE
              """,
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
            tabcmd publish "$wb" `
              --server   "$env:TC_SERVER_URL" `
              --site     "$env:TC_SITE_ID" `
              --token-name   "$env:TC_TOKEN_NAME" `
              --token-value  "$env:TC_TOKEN_SECRET" `
              --overwrite
          }
        '''
      }
    }
  }

  post {
    success {
      powershell '''
        $sha = (git rev-parse HEAD).Trim()
        $body = @{
          state       = "success"
          context     = "wiiisdom-tests"
          description = "All Wiiisdom tests passed. Published to Tableau Cloud."
        } | ConvertTo-Json
        Invoke-RestMethod `
          -Uri "https://api.github.com/repos/$env:REPO/statuses/$sha" `
          -Method POST `
          -Headers @{ Authorization="token $env:GITHUB_TOKEN"; "Content-Type"="application/json" } `
          -Body $body
      '''
    }

    failure {
      script {
        def details = env.FAILURE_DETAILS ?: 'Pipeline failed — check Jenkins build logs for details.'
        powershell """
          \$sha = (git rev-parse HEAD).Trim()
          \$body = @{
            state       = "failure"
            context     = "wiiisdom-tests"
            description = "Wiiisdom tests FAILED. Merge blocked."
          } | ConvertTo-Json
          Invoke-RestMethod \`
            -Uri "https://api.github.com/repos/\$env:REPO/statuses/\$sha" \`
            -Method POST \`
            -Headers @{ Authorization="token \$env:GITHUB_TOKEN"; "Content-Type"="application/json" } \`
            -Body \$body
        """

        if (env.CHANGE_ID) {
          def comment = """## Wiiisdom Test Results: FAILED

The following workbooks failed their Wiiisdom tests:

${details}

**This PR has NOT been merged.** Please fix the failing tests and push again.
          """
          powershell """
            \$body = @{ body = @'
${comment}
'@ } | ConvertTo-Json
            Invoke-RestMethod \`
              -Uri "https://api.github.com/repos/\$env:REPO/issues/\$env:CHANGE_ID/comments" \`
              -Method POST \`
              -Headers @{ Authorization="token \$env:GITHUB_TOKEN"; "Content-Type"="application/json" } \`
              -Body \$body
          """
        }
      }
    }
  }
}
