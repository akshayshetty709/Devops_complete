pipeline{
  agent any

  environment {
    S3_BUCKET = 'locate360-apk-storage'
    APK_PATH = 'android/app/build/outputs/apk/release/app-release.apk'
    }

  stages{
    stage('1.Checkout'){
      steps{
        git branch: 'Native-dev', credentialsId: 'github-cred', url: 'https://github.com/Locate360-Repo/Locate360-User-app.git'
      }
    }

    stage('2. npm install'){
      steps{
        sh "npm install"
      }
    }

    stage('3. prebuild'){
      steps{
        sh "npx expo prebuild"
      }
    }

    stage('4. build apk file'){
      steps{
          sh '''
          cd android &&
          chmod +x gradlew
          ./gradlew assembleRelease
          '''
      }
    }

    stage('5. Upload APK to S3') {
      steps {
        sh """
        aws s3 cp ${APK_PATH} \
        s3://${S3_BUCKET}/releases/Locate360-${BUILD_NUMBER}.apk

        aws s3 cp ${APK_PATH} \
        s3://${S3_BUCKET}/latest/Locate360-latest.apk
        """
      }
    }
  }
}
