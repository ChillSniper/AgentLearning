# 基础镜像
FROM openjdk:17-jdk-slim

# 作者
MAINTAINER xiaofuge

# 配置
ENV PARAMS=""

# 时区
ENV TZ=PRC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 添加应用
ADD target/ai-rag-knowledge-app.jar /ai-rag-knowledge-app.jar

ENTRYPOINT ["sh","-c","java -jar $JAVA_OPTS /ai-rag-knowledge-app.jar $PARAMS"]