# replace with your centos image
FROM harbor.jdts.com/centos/centos:7.4.1708
RUN touch /block1
RUN touch /block2
RUN touch /block3
RUN mkdir -p /data1
RUN mkdir -p /data2
RUN mkdir -p /data3

COPY sleep.sh /sleep.sh

RUN chmod +x /sleep.sh
ENTRYPOINT ["/sleep.sh"]