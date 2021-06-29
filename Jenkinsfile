#!/usr/bin/env groovy
pipeline {

  agent any

  triggers {  // Every day at 2:30
       cron('30 2 * * *')
  }

  stages {

    stage('Review content') {

      // The following review elements review the raw content, and do not require the site to be built first
      parallel {

//        stage('Forbid pDXC acronym') {
//          steps {
//            script {
//              // Forbid pDXC acronym in the documentation
//              echo 'Check for pDXC acronym in the documentation...'
//              sh 'rm -fR _site'
//              def pdxcCount = sh (returnStdout: true, script: 'find . -name "*.md" | xargs grep -i " pdxc[ .,-;:\\!\\?]" | wc -l').trim() as Integer
//              if (pdxcCount > 0) {
//                def pdxcOccurrences = sh (returnStdout: true, script: 'find . -name "*.md" | xargs grep -i " pdxc[ .,-;:\\!\\?]"').trim() as String
//                error("We prefer to use Platform DXC instead of pDXC in the documentation. Please check and correct: $pdxcOccurrences")
//              }
//            }
//          }
//        }

        stage('Spell Check') {
          agent {
          dockerfile {
              filename 'Dockerfile.mdspell'
              args '-u="root" -v $WORKSPACE:/srv/jekyll -w /srv/jekyll'
              reuseNode true
            }
          }
          environment {
            // mdspell uses chalk to color output.
            // chalk uses a library called supports-color which auto-detects terminal support.
            // this env var will force the library to use color.
            FORCE_COLOR = "1";
          }
          steps {
             echo 'Checking spelling...'
             sh  '''
             mdspell -V
             mdspell -n -a -r --en-us --dictionary dicts/en_US-large "*.md" "*/*.md" "*/*/*.md"
             '''
          }
        }
      }
    }

    // Jekyll takes the raw content and translates it into HTML output that can be published
    stage('Build site') {
      agent {
        dockerfile {
          filename 'Dockerfile.jekyll'
          args '-u="root" -v $WORKSPACE:/srv/jekyll -w /srv/jekyll'
          reuseNode true
        }

      }
      steps {
        echo 'Building site...'
        sh '''
        rm -fR _site
        bundle install
        bundle update
        bundle exec jekyll build --config _config.yml,_config_htmlproofer.yml
        '''
      }
    }

    // This stage reviews the resulting HTML output after the build, so must be run after Jekyll
    stage('HTML Proofer') {
      agent {
        dockerfile {
          filename 'Dockerfile.htmlproofer'
          args '-u="root" -v $WORKSPACE:/srv/jekyll -w /srv/jekyll'
          reuseNode true
        }
      }
      steps {

        // URL Ignore
        // - "edit/master" which are used to link back to live GH site to edit the raw Markdown
        // - "issues/new" which are used to generate new GitHub issues against the repo
        // - "jenkins.platformdxc.com", due to a CA certificate issue, but it requires authn anyway so wouldn't help to proof
        // - "/apis", "/use_cases" - related to deep permalinks under architecture section, issue #127

        echo 'Running HTML Proofer...'
        //   sh  'htmlproofer --allow-hash-href --empty-alt-ignore  --http-status-ignore "429"  --url-ignore /edit\\/master/,/issues\\/new/,/jenkins.platformdxc.com/,/\\/apis$/,/\\/use_cases$/ ./_site'
        sh  'htmlproofer --allow-hash-href --only-4xx --empty-alt-ignore  --http-status-ignore "429" --url-ignore /edit\\/master/,/issues\\/new/,/jenkins.platformdxc.com/,/\\/apis$/,/\\/use_cases$/ ./_site'

      }
    }
    // end HTML Proofer stage

  }
}
