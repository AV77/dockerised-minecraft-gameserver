FROM eclipse-temurin:25-jre-alpine

# Install dependencies: curl/jq for API fetching, su-exec for privilege dropping, bash for scripting
RUN apk add --no-cache su-exec bash curl jq

# Create an unprivileged user and group (UID/GID 1000)
RUN set -e <<-'EOF'
  addgroup -g 1000 minecraft
  adduser -D -u 1000 -G minecraft -h /data minecraft
EOF

# Set default environment variables
ENV MC_VERSION="26.1.1" \
    EULA="false" \
    JAVA_OPTS="-Xms2G -Xmx2G"

# Create the startup interceptor script
COPY --chmod=755 <<-'EOF' /start.sh
	#!/bin/bash
	if [ "$EULA" = "true" ] || [ "$EULA" = "TRUE" ]; then echo "eula=true" > /data/eula.txt; else echo "ERROR: You must accept the EULA by setting EULA=true in docker-compose.yml"; exit 1; fi

  # TODO: Update to use the new API
	if [ ! -f /data/server.jar ]; then
	  echo "Fetching latest PaperMC build for ${MC_VERSION}..."
	  LATEST_BUILD=$(curl -s https://api.papermc.io/v2/projects/paper/versions/${MC_VERSION}/builds | jq -r ".builds[-1].build")
	  JAR_NAME=$(curl -s https://api.papermc.io/v2/projects/paper/versions/${MC_VERSION}/builds/${LATEST_BUILD} | jq -r ".downloads.application.name")
	  echo "Downloading build ${LATEST_BUILD}..."
	  curl -o /data/server.jar "https://api.papermc.io/v2/projects/paper/versions/${MC_VERSION}/builds/${LATEST_BUILD}/downloads/${JAR_NAME}"
	fi
	
	echo "Adjusting volume permissions..."
	chown -R minecraft:minecraft /data
	
	# The execution interceptor
	if [ "$1" = 'java' ]; then
	    shift
	    echo "Starting server..."
	    exec su-exec minecraft java $JAVA_OPTS "$@"
	fi
	
	exec su-exec minecraft "$@"
EOF

# Set working directory to the mounted volume
WORKDIR /data

# Expose default Minecraft ports
EXPOSE 25565/tcp
EXPOSE 25565/udp

# Run the startup script
ENTRYPOINT ["/start.sh"]

# The perfectly clean CMD
CMD ["java", "-jar", "/data/server.jar", "nogui"]