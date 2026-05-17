FROM alpine:latest

# --- Build-Time Setup ---
# Run installation scripts in a single layer
RUN set -e <<-'EOF'
	# Install dependencies
	apk add --no-cache restic build-base git

	# Download and compile mcrcon
	git clone https://github.com/Tiiffi/mcrcon.git /tmp/mcrcon
	cd /tmp/mcrcon
	make
	make install

	# Clean up build files to keep the final image incredibly small
	rm -rf /tmp/mcrcon
	apk del build-base git
EOF

# Create the hot-backup script
COPY --chmod=755 <<-'EOF' /usr/local/bin/run_backup.sh
	#!/bin/sh
	echo "Pausing world saves..."
	mcrcon -H papermc -p "$RCON_PASSWORD" "save-off" "save-all flush"

	echo "Running restic backup..."
	restic backup /data --tag cron_backup

	echo "Resuming world saves..."
	mcrcon -H papermc -p "$RCON_PASSWORD" "save-on"
EOF

# Create the run-time setup script
COPY --chmod=755 <<-'EOF' /entrypoint.sh
	#!/bin/sh
	if ! restic snapshots >/dev/null 2>&1; then
		echo "Initializing new Restic repository..."
		restic init
	fi
	exec "$@"
EOF

# Set up cron jobs (Backup hour, Prune daily at 3 AM)
RUN set -e <<-'EOF'
	echo "0 * * * * /usr/local/bin/run_backup.sh" > /etc/crontabs/root
	echo "0 3 * * * restic forget --keep-last 48 --keep-daily 14 --keep-weekly 9999 --prune --tag cron_backup" >> /etc/crontabs/root
EOF

# Run-Time initialization
ENTRYPOINT ["/entrypoint.sh"]

# Keep alive via foreground cron daemon
CMD ["crond", "-f", "-d", "8"]