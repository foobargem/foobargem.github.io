---
title: Vault 를 사용한 TrustedUserCAKeys 기반 SSH 인증 구현
date: 2020-10-22T12:43:18+09:00
draft: false
author: foobargem
tags: ['vault', 'ssh', 'linux']
---

LDAP 또는 Kerberos 같은 통합인증(Single Sign-On)을 사용하지 않는
환경이라면 password 인증 또는 authorized_keys 인증을 주로 사용할 것이다.

팀 구성원에 변화(입사/퇴사)가 생기는 경우 password 를 변경하거나,
사용자의 public key 를 authorized_keys 에서 제거 또는 추가를 해야 한다.
ansible 같은 도구를 통해 관리를 해 나갈수 있지만 쉽지만은 않다.
게다가 인프라가 지속적으로 늘어나기에 이상적으로 관리하는 것은 쉽지 않은것 같다.

그래서 보통은 private key 를 공유해서 사용하는 경우가 빈번한것 같다.


## CA(Certificate Authority) 기반 인증

CA 서버를 통해서 각 사용자의 public key 에 대한 서명된 인증서를 발급받아 SSH 인증에 사용하는 방법이다. 서명된 인증서 발급시
인증 개체(user 또는 hostname), 유효기간 등을 설정해 줄수 있어서 사용자나 그룹별로 다른 권한과 유효기간 설정이 가능하게 된다.
Vault 를 CA 서버로 사용하면 `role` 이라는 설정이 있기 때문에 관리가 매우 편리해진다.

![vault role](/images/2020/10/vault_role.png)

## 인증 환경 준비

CA 서버에서 ssh-keygen 명령으로 keypair 를 생성한다.

`-C` 옵션은 코멘트, 
`-f` 옵션은 keyfile 의 위치및 이름이다.

자세한 내용은 `man ssh-keygen` 을 참고한다.


```
# ssh-keygen -C CA -f ca
...

# ls
ca  ca.pub
```

이제 모든 인프라에 `ca.pub` 과 `sshd_config` 를 배포한다.
public key 를 배포 하는 것이기 때문에 암호화가 적용되지 않은 통신 환경이라도 부담은 없다고 할수 있다.


/etc/ssh/sshd_config

```
TrustedUserCAKeys /etc/ssh/ca.pub
```

ansible 같은 도구를 사용하여 배포하는 것이 여러모로 유익하다.


## 수동으로 사용자의 public key 에 대해 인증해보기

Vault 로 CA 서버를 대체하기 전에 수동으로 인증을 시도해본다.
먼저 사용자의 public key(id_ecdsa.pub) 을 CA 서버로 복사한다.
역시 public key 이기 때문에 보안에 민감하지 않다.

CA 서버에서 사용자의 public key 를 가지고 인증서ID 는 myid, principals (계정명, 호스트명) 은 debian 으로,
유효기간은 현재시점 +10분으로, 일련번호는 2020102701 로 설정해서 인증서 발급을 한다.

```
debian@debug-gw:~/work/ssh_ca$ ssh-keygen -s ca -I myid -n debian -V +10m -z $DT user_pubkeys/id_ecdsa.pub
Signed user key user_pubkeys/id_ecdsa-cert.pub: id "myid" serial 20201028 for debian valid from 2020-10-28T15:05:00 to 2020-10-28T15:16:41

debian@debug-gw:~/work/ssh_ca$ ssh-keygen -Lf user_pubkeys/id_ecdsa-cert.pub
user_pubkeys/id_ecdsa-cert.pub:
        Type: ecdsa-sha2-nistp256-cert-v01@openssh.com user certificate
        Public key: ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ
        Signing CA: RSA SHA256:BnKkQ5ETL0CKE7rV1OBSPbBxDE0e7ObnztLJmP9IU5s
        Key ID: "myid"
        Serial: 20201027
        Valid: from 2020-10-27T15:36:00 to 2020-10-27T15:47:28
        Principals:
                debian
        Critical Options: (none)
        Extensions:
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
                permit-pty
                permit-user-rc
```

이제 인증서를 사용자 머신의 `.ssh/id_ecdsa-cert.pub` 으로 복사를 하고 private key 와 인증서를 가지고 인증을 시도해 본다.

### 인증서 만료 전

```
debian@dkr-deb10:~$ date
Tue 27 Oct 2020 03:46:27 PM JST
debian@dkr-deb10:~$ ssh -vv -i .ssh/id_ecdsa -i .ssh/id_ecdsa-cert.pub 192.168.0.63
...
debug1: Will attempt key: .ssh/id_ecdsa ECDSA SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug1: Will attempt key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug1: Will attempt key: .ssh/id_ecdsa-cert.pub ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: pubkey_prepare: done
...
debug2: service_accept: ssh-userauth
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug1: Authentications that can continue: publickey
debug1: Next authentication method: publickey
debug1: Offering public key: .ssh/id_ecdsa ECDSA SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: we sent a publickey packet, wait for reply
debug1: Authentications that can continue: publickey
debug1: Offering public key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: we sent a publickey packet, wait for reply
debug1: Server accepts key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: sign_and_send_pubkey: using private key ".ssh/id_ecdsa" for certificate
debug1: Authentication succeeded (publickey).
...
```

/var/log/auth.log 에는 다음와 같은 로그가 기록된다.

```
Oct 27 15:46:30 debian10 sshd[29783]: Accepted publickey for debian from 192.1
68.0.66 port 39344 ssh2: ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9U
qrGroLUQ ID myid (serial 20201027) CA RSA SHA256:BnKkQ5ETL0CKE7rV1OBSPbBxDE0e7
ObnztLJmP9IU5s
Oct 27 15:46:30 debian10 sshd[29783]: pam_unix(sshd:session): session opened f
or user debian by (uid=0)
```

### 인증서 만료 후

```
debian@dkr-deb10:~$ date
Tue 27 Oct 2020 03:48:21 PM JST

debian@dkr-deb10:~$ ssh -vv -i .ssh/id_ecdsa -i .ssh/id_ecdsa-cert.pub 192.168.0.63
...
debug1: Will attempt key: .ssh/id_ecdsa ECDSA SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug1: Will attempt key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug1: Will attempt key: .ssh/id_ecdsa-cert.pub ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: pubkey_prepare: done
...
debug1: Authentications that can continue: publickey
debug1: Next authentication method: publickey
debug1: Offering public key: .ssh/id_ecdsa ECDSA SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: we sent a publickey packet, wait for reply
debug1: Authentications that can continue: publickey
debug1: Offering public key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: we sent a publickey packet, wait for reply
debug1: Authentications that can continue: publickey
debug1: Offering public key: .ssh/id_ecdsa-cert.pub ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: we sent a publickey packet, wait for reply
debug1: Authentications that can continue: publickey
debug2: we did not send a packet, disable method
debug1: No more authentication methods to try.
debian@192.168.0.63: Permission denied (publickey).
```

/var/log/auth.log 에는 다음과 같은 로그가 기록된다.

```
Oct 27 15:48:23 debian10 sshd[29796]: error: Certificate invalid: expired
Oct 27 15:48:23 debian10 sshd[29796]: error: Certificate invalid: expired
Oct 27 15:48:23 debian10 sshd[29796]: Connection closed by authenticating user debian 192.168.0.66 port 39402 [preauth]
```

번거롭게 느껴질수 있지만 인증서 발급 로그와 인증 로그를 수집하여 실시간 모니터링 또는 사후 모니터링을 통해 사용자별 접근 기록을 관리할 수 있다.
또한 인증서에 만료시간, 할당된 시스템 계정, ssh 확장기능의 정보를 설정할수 있기 때문에 사용자별로 차별을 줄 수 있다.


이미 발급된 인증서에 만료시간이 있기 때문에 퇴사한 멤버가 악의적인 SSH 접근이 불가능하다.
그래서 모든 인프라의 authorized_keys 에서 퇴사한 멤버의 public key 를 제거하는 수고를 없앨 수 있다.

그런데, 이 과정을 자동화 하려면 뭔가 만들어야 할것 같지만
Vault 와 `Signed SSH Certificates` Secrets Engine 을 사용하면 뭔가 만들 필요가 없어진다.

간단하게 Dev 환경으로 Vault 를 구동하여 동작을 확인해 본다.


## Dev 환경으로 vault 기동

Docker 와 docker-compose 가 필요하다.

docker-compose.yml

```
version: '3.8'
services:
    dev-vault:
        image: vault
        container_name: dev-vault
        cap_add:
        - IPC_LOCK
        ports:
        - "8200:8200"
```

**IPC_LOCK** 은 container 로 구동된 Vault 에 저장되는 민감한 데이터가 Host 로 swap 되는 것을 방지하기 위함이다.
(참고: [https://hub.docker.com/_/vault](https://hub.docker.com/_/vault))

docker-compose 로 container 구동을 한다.

```
$ docker-compose up -d
```

web 기반 UI 접속을 위해 docker-compose logs 에 남겨진 **Root Token** 값을 확인한다.

```
$ docker-compose logs | grep 'Root Token'
dev-vault    | Root Token: s.WVYvs3DZDUU7w1iN8PjtB0PN
```


## Secret engine 설정

`Secrets` 페이지에서 **Enable new engine +** 을 클릭하고 **SSH** 를 선택한다.

![vault enable engine](/images/2020/10/vault_enable_ssh_engine.png)

Path 는 적당하게 설정한다.
본 예제는 **ssh-client-signer** 라고 설정했다.

Method options 는 기본값으로 두고 **Enable engine** 을 클릭하면 **ssh-client-signer** 라는 path 를 갖는 SSH secret engine 이 활성화 된다.

이제는 인증서 발급에 사용할 keypair 를 설정해야 한다.
Configuration 탭을 클릭하고 **Configure** 를 클릭한다.

![vault configure ssh1](/images/2020/10/vault_ssh_engine_show.png)

미리 만들어둔 keypair(ca, ca.pub) 의 내용을 기입한다.
만약 새로 만들고 싶다면 **Generate signing key** 를 체크한다.
기존에 만든것을 사용하고 싶다면 체크해제 한다.

![vault configure ssh2](/images/2020/10/vault_configure_ssh.png)


## Role 설정

우측의 **Create role +** 을 클릭한뒤 적절한 값을 작성한다.

![vault create role](/images/2020/10/vault_create_role.png)

* Role name: sre
* Key type: ca
* Allow user certificates: 선택
* Default Username: debian
* Allowed users: debian
* TTL: 600초 (10분)

**Default extensions: permit-pty 를 추가**

```
{
    "permit-pty": ""
}
```

이것으로 인증서 발급을 위한 작업은 끝났다.
하지만 누구에게 어떤 role 을 줄것인지 접근제어를 설정하는것이 남았다.


## 접근제어

먼저 ACL Policy 를 추가한다.
Policies 페이지에서 **Create ACL policy +** 를 클릭한다.

name 은 **ssh-user-sign** 으로 policy 는 다음과 같이 기입한다.

```
path "ssh-client-signer/config/ca" {
    capabilities = ["read"]
}
path "ssh-client-signer/sign/sre" {
    capabilities = ["read", "update"]
}
```

이제는 사용자의 인증을 위한 Access 설정을 한다.

**Access** 페이지를 클릭하여 **Authentication Methods** 메뉴의 **Enable new method +** 를 클릭하고 연동할 인증 시스템을 선택한다.
본 예제는 **Username &amp; Password** 를 선택했다.

생성된 **userpass/** 를 클릭하고 **Create user +** 를 클릭한다.
Username 과 Password 를 기입하고 **Tokens** 버튼을 눌러
**Generated Token’s Policies** 에 **ssh-user-sign** 을 기입후 **Add** 버튼을 클릭한다.

![vault token policy](/images/2020/10/vault_token_policy.png)

모든 준비가 끝났다.


## Vault 인증후 public key 에 대한 인증서 발급받기

사용자 환경에 vault 를 다운로드 받고 적당한 PATH 에 옮겨둔다.

**VAULT_ADDR** 환경변수를 선언해둔다.

```
$ export VAULT_ADDR=http://192.168.0.66:8200
```

Vault 에 로그인 시도를 하여 인증이 되면 token 발급이 된다.


```
$ vault login -method=userpass username=toast password='cloud'
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  s.g2NQFk3c6YK182z6Z44yywQF
token_accessor         uRzcznpkjNg6EbPUzEuVDyTc
token_duration         768h
token_renewable        true
token_policies         ["default" "ssh-user-sign"]
identity_policies      []
policies               ["default" "ssh-user-sign"]
token_meta_username    toast
```

SSH 로그인을 위해 public key 에 대한 인증서 발급을 요청해본다.

```
$ vault write ssh-client-signer/sign/sre \
    public_key=@$HOME/.ssh/id_ecdsa.pub
ecdsa-sha2-nistp256-cert-v01@openssh.com AAAAKG
C12MDFAb3BlbnNzaC5jb20AAAAgay/zSvnv97yG2/9i+Fjp
...
RgJCkTFFbPNKPfxj2D4CvmA==
```

인증서 발급이 되면 바로 **$HOME/.ssh/id_ecdsa-cert.pub** 에 저장되도록 한다.

```
$ vault write --field=signed_key ssh-client-signer/sign/sre \
    public_key=@$HOME/.ssh/id_ecdsa.pub > $HOME/.ssh/id_ecdsa-cert.pub
```

발급된 인증서 정보를 확인해 본다.

```
$ ssh-keygen -Lf ~/.ssh/id_ecdsa-cert.pub
/home/debian/.ssh/id_ecdsa-cert.pub:
        Type: ecdsa-sha2-nistp256-cert-v01@openssh.com user certificate
        Public key: ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ
        Signing CA: RSA SHA256:BnKkQ5ETL0CKE7rV1OBSPbBxDE0e7ObnztLJmP9IU5s
        Key ID: "vault-userpass-toast-5f16d4e72beb7f0968d88fa7a3bf2d4bee78a94a91dff9fa2fd52aac6ae82d44"
        Serial: 7565235238969908896
        Valid: from 2020-10-28T15:31:56 to 2020-10-28T15:42:26
        Principals:
                debian
        Critical Options: (none)
        Extensions:
                permit-pty
```

이제 리모트 SSH 접속을 시도해 본다.

```
$ ssh -vv -i .ssh/id_ecdsa 192.168.0.63
...
debug1: Will attempt key: .ssh/id_ecdsa ECDSA SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug1: Will attempt key: .ssh/id_ecdsa ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ explicit
debug2: pubkey_prepare: done
...
```

auth.log 에는 다음과 같은 기록이 확인된다.

```
Oct 28 15:36:27 debian10 sshd[30265]: Accepted publickey for debian from 192.168.0.66 port 56696 ssh2: ECDSA-CERT SHA256:XxbU5yvrfwlo2I+no78tS+54qUqR3/n6L9UqrGroLUQ ID vault-userpass-toast-5f16d4e72beb7f0968d88fa7a3bf2d4bee78a94a91dff9fa2fd52aac6ae82d44 (serial 7565235238969908896) CA RSA SHA256:BnKkQ5ETL0CKE7rV1OBSPbBxDE0e7ObnztLJmP9IU5s
```


## 정리

Vault 가 만능은 아니겠지만 CI/CD 환경이 늘어나고 클라우드 인프라가 지속적으로 증가하는 환경에서 민감한 데이터를 관리하는 도구로 매력적임은 분명하다.
개념과 배우기가 어려울뿐… :)

## 참조

* [https://engineering.fb.com/security/scalable-and-secure-access-with-ssh/](https://engineering.fb.com/security/scalable-and-secure-access-with-ssh/)
* [https://www.vaultproject.io/docs/secrets/ssh/signed-ssh-certificates](https://www.vaultproject.io/docs/secrets/ssh/signed-ssh-certificates)
