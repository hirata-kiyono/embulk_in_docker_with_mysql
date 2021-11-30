FROM openjdk:8-slim

ENV LANG=C.UTF-8 \
    PATH_TO_EMBULK=/opt/embulk \
    PATH=${PATH}:/opt/embulk

# Change timezone
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

RUN apt-get update && apt-get install -y curl vim

# Embulk install
RUN mkdir -p ${PATH_TO_EMBULK} \
    && curl --create-dirs -o ${PATH_TO_EMBULK}/embulk -L "https://dl.embulk.org/embulk-0.9.23.jar" \
    && chmod +x ${PATH_TO_EMBULK}/embulk

RUN embulk gem install embulk-output-mysql

WORKDIR /app
#COPY ./app/load.yml.liquid ./

CMD [ "bash" ]