FROM ubuntu:18.04
RUN dpkg --add-architecture i386 && \
    apt-get update -y && \
    apt-get install -y --no-install-recommends libncurses5:i386 libc6:i386 libstdc++6:i386 lib32gcc1 lib32ncurses5 lib32z1 zlib1g:i386 && \
    apt-get install -y --no-install-recommends openjdk-8-jdk && \
    apt-get install -y --no-install-recommends git wget unzip && \
    apt-get install -y --no-install-recommends qt5-default

ARG GRADLE_VERSION=5.2.1
ARG GRADLE_DIST=bin
RUN cd /opt && \
    wget -q https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-${GRADLE_DIST}.zip && \
    unzip gradle*.zip && \
    ls -d */ | sed 's/\/*$//g' | xargs -I{} mv {} gradle && \
    rm gradle*.zip

# download and install Android SDK
# https://developer.android.com/studio/#downloads
ARG ANDROID_SDK_VERSION=4333796
ENV ANDROID_HOME /opt/android-sdk
RUN mkdir -p ${ANDROID_HOME} && cd ${ANDROID_HOME} && \
    wget -q https://dl.google.com/android/repository/sdk-tools-linux-${ANDROID_SDK_VERSION}.zip && \
    unzip *tools*linux*.zip && \
    rm *tools*linux*.zip
# set the environment variables
ENV APK_SIGNER=/opt/android-sdk/build-tools/29.0.2/
ENV JAVA_HOME /usr/lib/jvm/java-8-openjdk-amd64
ENV ANDROID_SDK_ROOT /opt/android-sdk
ENV GRADLE_HOME /opt/gradle
ENV PATH ${PATH}:${GRADLE_HOME}/bin:${ANDROID_HOME}/emulator:${ANDROID_HOME}/tools/bin:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools:${ANDROID_HOME}/tools/bin/apksigner:${APK_SIGNER}
ENV _JAVA_OPTIONS -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap
# WORKAROUND: for issue https://issuetracker.google.com/issues/37137213
ENV LD_LIBRARY_PATH ${ANDROID_HOME}/emulator/lib64:${ANDROID_HOME}/emulator/lib64/qt/lib


# accept the license agreements of the SDK components
ADD license_accepter.sh /opt/
RUN chmod +x /opt/license_accepter.sh && /opt/license_accepter.sh $ANDROID_SDK_ROOT

# setup adb server
EXPOSE 5037

# install and configure SSH server
EXPOSE 22
ADD sshd-banner /etc/ssh/
ADD authorized_keys /tmp/
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends openssh-server supervisor locales && \
    mkdir -p /var/run/sshd /var/log/supervisord && \
    locale-gen en en_US en_US.UTF-8 && \
    apt-get remove -y locales && apt-get autoremove -y && \
    FILE_SSHD_CONFIG="/etc/ssh/sshd_config" && \
    echo "\nBanner /etc/ssh/sshd-banner" >> $FILE_SSHD_CONFIG && \
    echo "\nPermitUserEnvironment=yes" >> $FILE_SSHD_CONFIG && \
    ssh-keygen -q -N "" -f /root/.ssh/id_rsa && \
    FILE_SSH_ENV="/root/.ssh/environment" && \
    touch $FILE_SSH_ENV && chmod 600 $FILE_SSH_ENV && \
    printenv | grep "JAVA_HOME\|GRADLE_HOME\|ANDROID_HOME\|LD_LIBRARY_PATH\|PATH" >> $FILE_SSH_ENV && \
    FILE_AUTH_KEYS="/root/.ssh/authorized_keys" && \
    touch $FILE_AUTH_KEYS && chmod 600 $FILE_AUTH_KEYS && \
    for file in /tmp/*.pub; \
    do if [ -f "$file" ]; then echo "\n" >> $FILE_AUTH_KEYS && cat $file >> $FILE_AUTH_KEYS && echo "\n" >> $FILE_AUTH_KEYS; fi; \
    done && \
    (rm /tmp/*.pub 2> /dev/null || true)

ADD supervisord.conf /etc/supervisor/conf.d/
ADD supervisord.conf /etc/supervisor/conf.d/
#ADD android.keystore /Android-APK/
#ADD app/build/outputs/apk/release/ /Android-APK/
#RUN sdkmanager "platform-tools" "platforms;android-29" "build-tools;29.0.2" "add-ons;addon-google_apis-google-24"

#RUN apksigner sign --ks /Android-APK/android.keystore --ks-pass pass:android --key-pass pass:android --in /Android-APK/app-release-unsigned.apk --out /Android-APK/signed-app.apk
#RUN zipalign -f 4 /Android-APK/signed-app.apk /Android-APK/aligned-app.apk
#RUN sdkmanager "system-images;android-23;google_apis;x86"
#RUN adb install /Android-APK/aligned-app.apk

RUN sdkmanager --sdk_root=$ANDROID_HOME --install \
  "platform-tools" \
  "build-tools;29.0.2" \
  "platforms;android-25" \
  "system-images;android-25;google_apis;arm64-v8a" \
  "emulator"

RUN avdmanager create avd --force --name "device" --abi "arm64-v8a" --package 'system-images;android-25;google_apis;arm64-v8a' --device "Nexus 10"
CMD ["/usr/bin/supervisord"]
