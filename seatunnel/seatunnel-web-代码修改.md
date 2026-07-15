●# 在仓库根目录执行
mkdir -p seatunnel-connectors && cd seatunnel-connectors

# 下载 3 个连接器 jar（Maven 中央仓库）
curl -L -o connector-jdbc-2.3.11.jar \
https://repo1.maven.org/maven2/org/apache/seatunnel/connector-jdbc/2.3.11/connector-jdbc-2.3.11.jar

curl -L -o connector-doris-2.3.11.jar \
https://repo1.maven.org/maven2/org/apache/seatunnel/connector-doris/2.3.11/connector-doris-2.3.11.jar

curl -L -o connector-cdc-mysql-2.3.11.jar \
https://repo1.maven.org/maven2/org/apache/seatunnel/connector-cdc-mysql/2.3.11/connector-cdc-mysql-2.3.11.jar

# 下载官方二进制包，只抽出 plugin-mapping.properties
curl -L -o /tmp/st.tar.gz \
https://dlcdn.apache.org/seatunnel/2.3.11/apache-seatunnel-2.3.11-bin.tar.gz
tar -xzf /tmp/st.tar.gz --strip-components=2 -C . \
apache-seatunnel-2.3.11/connectors/plugin-mapping.properties
rm /tmp/st.tar.gz