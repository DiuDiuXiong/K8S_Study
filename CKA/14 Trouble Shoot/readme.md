# Trouble Shoot
1. If it is issues related before pod/deployment start `kubectl describe ...`
2. If it is issues related after pod/deployment start `kubectl logs ...`
3. If it is control plane failures issue follow the follow path:
   - Verify Cluster Node Health: `kubectl get nodes`
   - Confirm Control Plane Component Status: `systemctl kube-apiserver status` && `systemctl kube-controller-manager status` && `systemctl kube-scheduler status`
   - Review Control Plane Logs: `kubectl logs kube-apiserver-master -n kube-system` && `sudo journalctl -u kube-apiserver`
   - `journalctl -u kubelet` to see kubelet log if kubelet is down
   - `systemctl <bin command> [status|start|restart]`
4. For debug network issues:
   https://rudimartinsen.com/2021/01/14/cka-notes-troubleshooting/

Finally, for some advanced json kubectl commands:

| Command                                      | Description                                  | Example                                                              |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------- |
| Custom Columns                               | Display specific columns                     | `kubectl get nodes -o=custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu` |
| Sorting by Name                              | Sort nodes alphabetically                  | `kubectl get nodes --sort-by=.metadata.name`                         |
| Sorting by CPU                               | Sort nodes by CPU capacity                   | `kubectl get nodes --sort-by=.status.capacity.cpu`                  |