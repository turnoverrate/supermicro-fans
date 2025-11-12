# Supermicro IPMI Fan Control

A smart, automated fan control script for Supermicro servers with IPMI support. This script monitors CPU temperatures and dynamically adjusts fan speeds to maintain optimal cooling while minimizing noise.

## Features

- **Ultra-Quiet Operation**: Maintains 15% fan speed for normal temperatures (below 70°C)
- **Temperature-Based Control**: Automatically adjusts fan speeds based on CPU temperature
- **Dual-Zone Management**: Controls both CPU fans (Zone 0) and peripheral fans (Zone 1) independently
- **Safety First**: Multiple safety mechanisms including emergency thresholds and automatic failover
- **Systemd Integration**: Runs as a service with automatic startup
- **Detailed Logging**: Comprehensive logs with automatic rotation

## Hardware Requirements

- Supermicro server motherboard with IPMI support
- IPMI accessible via `ipmitool`
- Linux-based operating system

## Software Requirements

- `ipmitool` - IPMI management utility
- `bash` - Shell scripting environment
- Root/sudo access

## Quick Start

### 1. Install Dependencies

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install ipmitool

# RHEL/CentOS/Rocky
sudo yum install ipmitool
```

### 2. Install the Script

```bash
# Copy the script to system location
sudo cp supermicro-fan-control.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/supermicro-fan-control.sh

# Install the systemd service
sudo cp supermicro-fan-control.service /etc/systemd/system/
sudo systemctl daemon-reload
```

### 3. Test the Script

Before enabling as a service, test manually:

```bash
# Run the script in foreground to verify it works
sudo /usr/local/bin/supermicro-fan-control.sh

# Watch the output for a few minutes
# Press Ctrl+C to stop
```

### 4. Enable the Service

```bash
# Enable and start the service
sudo systemctl enable supermicro-fan-control.service
sudo systemctl start supermicro-fan-control.service

# Check status
sudo systemctl status supermicro-fan-control.service

# Monitor logs
sudo tail -f /var/log/fan-control.log
```

## Fan Curve Configuration

### Current Default: Ultra-Quiet Profile

The script uses an ultra-quiet fan curve optimized for home/office environments:

| Temperature Range | Fan Speed | Notes |
|-------------------|-----------|-------|
| < 70°C            | 15%       | Almost silent - normal operation |
| 70-75°C           | 60%       | High load cooling |
| 75-80°C           | 70%       | Very high load |
| 80-85°C           | 80%       | Near emergency threshold |
| 85-90°C           | 90%       | Critical temperatures |
| ≥ 90°C            | 100%      | Maximum cooling |

**Emergency Threshold**: 95°C (triggers 100% fan speed and safety shutdown)

### Customizing the Fan Curve

Edit the `FAN_CURVE` array in [supermicro-fan-control.sh](supermicro-fan-control.sh) (lines 41-55):

#### Alternative: Balanced Profile

For more gradual temperature response:

```bash
declare -A FAN_CURVE=(
    [0]=15      # Below 40°C: 15%
    [40]=20     # 40-50°C: 20%
    [50]=30     # 50-60°C: 30%
    [60]=45     # 60-70°C: 45%
    [70]=60     # 70-75°C: 60%
    [75]=75     # 75-80°C: 75%
    [80]=90     # 80-85°C: 90%
    [85]=100    # Above 85°C: 100%
)
```

#### Alternative: Performance Profile

For maximum cooling and lower temperatures:

```bash
declare -A FAN_CURVE=(
    [0]=25      # Below 35°C: 25%
    [35]=30     # 35-45°C: 30%
    [45]=40     # 45-55°C: 40%
    [55]=55     # 55-65°C: 55%
    [65]=70     # 65-75°C: 70%
    [75]=85     # 75-80°C: 85%
    [80]=100    # Above 80°C: 100%
)
```

After modifying the fan curve:

```bash
sudo systemctl restart supermicro-fan-control.service
sudo tail -f /var/log/fan-control.log
```

## Configuration Options

Edit [supermicro-fan-control.sh](supermicro-fan-control.sh) to customize:

| Setting | Default | Description |
|---------|---------|-------------|
| `LOG_FILE` | `/var/log/fan-control.log` | Log file location |
| `MAX_LOG_SIZE` | `10485760` (10MB) | Maximum log size before rotation |
| `PRIMARY_SENSORS` | `("CPU1 Temp" "CPU2 Temp")` | Temperature sensors to monitor |
| `FAN_ZONES` | `(0 1)` | Fan zones to control |
| `POLL_INTERVAL` | `10` | Temperature check interval (seconds) |
| `EMERGENCY_TEMP` | `95` | Emergency shutdown temperature (°C) |

## Safety Features

1. **Emergency Threshold Protection**: If temps exceed 95°C, fans go to 100% and script exits to auto mode
2. **Sensor Failure Protection**: If sensors fail to read, fans go to 100% and revert to auto mode
3. **Service Stop Safety**: When service stops, automatic fan control is restored
4. **Dual-Zone Redundancy**: Both zones controlled independently for reliability
5. **Conservative Design**: 15% minimum fan speed ensures adequate airflow at all times
6. **Log Rotation**: Automatic log rotation prevents disk space issues

## Monitoring

### View Current Status

```bash
# Check service status
sudo systemctl status supermicro-fan-control.service

# View recent logs
sudo journalctl -u supermicro-fan-control.service -n 50

# Monitor live logs
sudo tail -f /var/log/fan-control.log
```

### Check Temperatures and Fan Speeds

```bash
# View CPU temperatures
ipmitool sensor | grep -i temp

# View fan speeds (RPM)
ipmitool sensor | grep -i fan

# Monitor in real-time
watch -n 2 'ipmitool sensor | grep -E "(Temp|FAN)"'
```

### Example Log Output

```
[2025-11-12 10:30:00] [INFO] Starting main control loop (polling every 10 seconds)
[2025-11-12 10:30:00] [INFO] Controlling Zone 0 (CPU fans) and Zone 1 (Peripheral fans) independently
[2025-11-12 10:30:00] [INFO] Temperature: 45°C | Setting both zones: 15%
[2025-11-12 10:31:00] [INFO] Temperature: 45°C | Both zones: 15% (stable)
[2025-11-12 10:35:00] [INFO] Temperature: 72°C | Setting both zones: 60%
```

## Manual IPMI Control

### Enable Manual Fan Control

```bash
sudo ipmitool raw 0x30 0x45 0x01 0x01
```

### Set Fan Speed for Specific Zones

```bash
# Zone 0 (CPU fans) to 50% (0x7F = 127 = 50%)
sudo ipmitool raw 0x30 0x70 0x66 0x01 0x00 0x7F

# Zone 1 (Peripheral fans) to 50%
sudo ipmitool raw 0x30 0x70 0x66 0x01 0x01 0x7F
```

### Return to Automatic Mode

```bash
sudo ipmitool raw 0x30 0x01 0x01
```

### Hex Conversion Reference

| % | Hex | Decimal |
|---|-----|---------|
| 15% | 0x26 | 38 |
| 20% | 0x33 | 51 |
| 25% | 0x3F | 63 |
| 30% | 0x4C | 76 |
| 40% | 0x66 | 102 |
| 50% | 0x7F | 127 |
| 60% | 0x99 | 153 |
| 70% | 0xB2 | 178 |
| 80% | 0xCC | 204 |
| 90% | 0xE5 | 229 |
| 100% | 0xFF | 255 |

## Troubleshooting

### Service Won't Start

```bash
# Check for errors
sudo journalctl -u supermicro-fan-control.service -n 50

# Verify ipmitool works
sudo ipmitool sensor get "CPU1 Temp"

# Test manual fan control
sudo ipmitool raw 0x30 0x45 0x01 0x01
sudo ipmitool raw 0x30 0x70 0x66 0x01 0x00 0x33
```

### Fans Not Responding

```bash
# Enable manual mode
sudo ipmitool raw 0x30 0x45 0x01 0x01

# Test Zone 0 at 100%
sudo ipmitool raw 0x30 0x70 0x66 0x01 0x00 0xFF
sleep 3

# Test Zone 1 at 100%
sudo ipmitool raw 0x30 0x70 0x66 0x01 0x01 0xFF
sleep 3

# Return to auto mode
sudo ipmitool raw 0x30 0x01 0x01
```

### High CPU Usage

The script is designed to be lightweight and should use minimal CPU. If you see high usage:

```bash
# Check polling interval (default: 10 seconds)
grep POLL_INTERVAL /usr/local/bin/supermicro-fan-control.sh
```

### Script Exits Unexpectedly

Check logs for errors:

```bash
sudo tail -100 /var/log/fan-control.log
sudo journalctl -u supermicro-fan-control.service -n 100
```

Common causes:
- Temperature sensors become unreadable
- IPMI commands failing
- Emergency temperature threshold exceeded

## Expected Noise Levels

With the ultra-quiet default configuration:

| Fan Speed | Noise Level | When It Occurs |
|-----------|-------------|----------------|
| 15% | Almost silent | Normal operation (< 70°C) |
| 60% | Very loud | High load (70-75°C) |
| 70% | Very loud | Very high load (75-80°C) |
| 80% | Extremely loud | Near emergency (80-85°C) |
| 90% | Extremely loud | Critical (85-90°C) |
| 100% | Maximum | Emergency (≥ 90°C) |

## Uninstalling

```bash
# Stop and disable the service
sudo systemctl stop supermicro-fan-control.service
sudo systemctl disable supermicro-fan-control.service

# Remove files
sudo rm /usr/local/bin/supermicro-fan-control.sh
sudo rm /etc/systemd/system/supermicro-fan-control.service
sudo rm /var/log/fan-control.log*

# Reload systemd
sudo systemctl daemon-reload

# Return IPMI to automatic mode
sudo ipmitool raw 0x30 0x01 0x01
```

## How It Works

1. **Initialization**: Script enables manual fan control mode via IPMI
2. **Monitoring**: Every 10 seconds (configurable), reads CPU temperatures from both CPUs
3. **Calculation**: Uses the maximum temperature to determine required fan speed from the fan curve
4. **Control**: Sends IPMI commands to set both fan zones to the calculated speed
5. **Safety**: Continuously monitors for emergency conditions and sensor failures
6. **Cleanup**: On exit, returns fans to automatic IPMI control mode

## Architecture

- **Zone 0**: CPU fans (directly cooling processors)
- **Zone 1**: Peripheral/System fans (case airflow)
- **Control Strategy**: Both zones use the same fan curve for consistent airflow and noise levels
- **Temperature Source**: Maximum temperature from CPU1 and CPU2 sensors

## Contributing

Feel free to submit issues, fork the repository, and create pull requests for any improvements.

## License

See [LICENSE](LICENSE) file for details.

## Acknowledgments

- Built for Supermicro X10/X11 generation motherboards
- Inspired by the need for quieter home lab servers
- Uses IPMI raw commands for granular fan control

## Support

If you encounter issues:

1. Check the troubleshooting section above
2. Review logs: `/var/log/fan-control.log`
3. Test IPMI commands manually
4. Verify sensor readings with `ipmitool sensor`
5. Open an issue with details about your hardware and error messages

---

**Warning**: This script takes control of your server's fan speeds. While it includes multiple safety features, use at your own risk. Monitor temperatures carefully after installation to ensure proper cooling for your specific hardware and workload.
