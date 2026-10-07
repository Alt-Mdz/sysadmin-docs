EMERGENCY ROLLBACK & DISK RECOVERY PROCEDURE
Objective: To quickly revert critical volume capacity below 85% and restore service availability following a disk saturation event.
Execution Steps:
1. Identify the root cause: Locate the largest directories using:
du -ah / 2>/dev/null | sort -rh | head -n 20
2. Apply safe data purging: Execute package cache clearing and log truncation as detailed in the Emergency Cleandown Steps.(Politicas de gobernanza)
3. Strict Constraints: Do NOT manually delete application source code, databases (.db, .sql), or active .conf files without explicit approval from the Infrastructure Lead.
4. Service Restoration: Run df -h to verify clearance. Once verified, restart affected daemons using systemctl restart [service_name].