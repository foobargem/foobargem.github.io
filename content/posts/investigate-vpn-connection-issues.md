---
title: "Investigate VPN connection issues"
date: 2024-02-08T22:18:50+09:00
tags: ["aws", "openvpn"]
author: foobargem
---

지금 회사는 AWS Client VPN 을 사용하고 있고, 종종 VPN 연결이 안된다는 연락을 받고 있다.
대부분의 경우는 Client VPN endpoint 의 Connections 정보와 IAM Identity Center 에서 해당 유저의 Active sessions 를 점검하고, 설정 변경 여부를 확인하면서 해결을 할수 있었다.
이번에는 좀 특이한 상황이었기에 기록을 남겨둔다.

VPN Client 로 연결을 시도했더니 `Connection failed. Try again.` 이라는 오류가 나면서 연결이 안되고 있다고 슬랙으로 연락을 받았다.
![aws_vpn_client_connection_failed](/images/2024/aws_vpnclient_connection_failed.png)

처음에는 Identity Center 에서 해당 유저의 Active sessions 에 1개의 session 이 확인되어, 삭제하고 다시 연결을 해달라고 답을 했다.
하지만 그래도 연결이 안된다고 답이 왔다.

어떤 문제인지 단서가 없었기에 [AWS 의 가이드](https://docs.aws.amazon.com/ko_kr/vpn/latest/clientvpn-user/macos-troubleshooting.html) 를 참고하여 `aws_vpn_client_YYYYmmdd.log` 로그 파일을 요청했다.

```
2024-02-05 08:56:52.601 +05:30 [INF] Starting OpenVpn process
2024-02-05 08:56:52.601 +05:30 [DBG] Starting process
2024-02-05 08:56:52.612 +05:30 [DBG] Start to read process output
2024-02-05 08:57:05.219 +05:30 [DBG] End reading process output
2024-02-05 08:57:05.296 +05:30 [DBG] Helper app --init output: Kill failed.
2024-02-05 08:57:05.296 +05:30 [DBG] Stopping DNS monitoring thread
2024-02-05 08:57:05.296 +05:30 [DBG] Releasing DNS monitoring lock
2024-02-05 08:57:05.296 +05:30 [DBG] Metric agent started
2024-02-05 08:57:05.296 +05:30 [DBG] Sending Metrics batches
2024-02-05 08:57:05.296 +05:30 [DBG] Connection state changed for CVPN endpoint id: cvpn-endpoint-foobargem
2024-02-05 08:57:05.296 +05:30 [DBG] Received exception for connection state Disconnected. Show error message to user
2024-02-05 08:57:05.296 +05:30 [ERR] Exception recieved by connection view controller
ACVC.Core.OpenVpn.OvpnProcessStillRunningException: Kill failed
  at ACVC.Core.OpenVpn.OvpnOsxProcessManager.Start (System.String openVpnConfigPath, System.String managementPortPasswordFile, System.Int32 timeoutMilliseconds) [0x001d4] in <aa51f92ef5dd4672897da646a5393ec3>:0
  at ACVC.Core.OpenVpn.OvpnConnectionManager.Connect (ACVC.Core.Metadata.OvpnConnectionProfile configProfile, ACVC.Core.GetCredentialsCallback getCredentialsCallback, System.Int32 timeout) [0x00262] in <aa51f92ef5dd4672897da646a5393ec3>:0
2024-02-05 08:57:05.298 +05:30 [DBG] Checking if table MetricsTable is empty
2024-02-05 08:57:05.299 +05:30 [DBG] is empty: False
```

로그를 확인해보니 `ACVC.Core.OpenVpn.OvpnProcessStillRunningException` 이라는 exception 을 확인할수 있었고 해당 직원의 OS에는 vpn client 가 동작중인 상태임을 추측할수 있었다.
간단하게 해결할수 있도록 PC 를 재부팅 해달라 요청했고, 재부팅 한뒤 정상적으로 VPN 연결이 성공했다는 답을 받았다.

로그는 문제 해결에 확실한 단서를 제공해주기 때문에 로그에 남겨지는 메시지들도 의미있게 잘 작성되어야 함을 다시금 일깨워준 시간이었다.
