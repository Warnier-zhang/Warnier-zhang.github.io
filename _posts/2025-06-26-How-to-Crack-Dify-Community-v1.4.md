---
layout: post
title: 破解Dify 1.4社区版
---

和**Dify云服务**、**企业版**相比，**社区版**有点限制，如：无法创建工作空间、应用数量不能超过5个等等。本文总结了网上突破部分限制的做法，**仅供学习交流**，在预算充足的情况下，推荐使用Dify云服务！

#### 限制

##### 1、无法创建工作空间？

`api`和`worker`容器中的`feature_service.py`、`account_service.py`服务涉及到管理工作空间。考虑到`api`和`worker`容器都是基于`langgenius/dify-api`镜像运行的，先修改两者中的一个，再复制到另一个即可。

首先，从`api`容器中下载`feature_service.py`和`account_service.py`2个文件：

```
docker container cp dify-api-1:/app/api/services/feature_service.py feature_service.py
docker container cp dify-api-1:/app/api/services/account_service.py account_service.py
```

再，按如下内容编辑这2个文件

`feature_service.py`：

```
...
class SystemFeatureModel(BaseModel):
    sso_enforced_for_signin: bool = False
    sso_enforced_for_signin_protocol: str = ""
    enable_marketplace: bool = False
    max_plugin_package_size: int = dify_config.PLUGIN_MAX_PACKAGE_SIZE
    enable_email_code_login: bool = False
    enable_email_password_login: bool = True
    enable_social_oauth_login: bool = False
    is_allow_register: bool = False
    # 修改
    # is_allow_create_workspace: bool = False
    is_allow_create_workspace: bool = True
    is_email_setup: bool = False
    license: LicenseModel = LicenseModel()
    branding: BrandingModel = BrandingModel()
    webapp_auth: WebAppAuthModel = WebAppAuthModel()
...
class FeatureService:
...
	@classmethod
    def _fulfill_system_params_from_env(cls, system_features: SystemFeatureModel):
        system_features.enable_email_code_login = dify_config.ENABLE_EMAIL_CODE_LOGIN
        system_features.enable_email_password_login = dify_config.ENABLE_EMAIL_PASSWORD_LOGIN
        system_features.enable_social_oauth_login = dify_config.ENABLE_SOCIAL_OAUTH_LOGIN
        system_features.is_allow_register = dify_config.ALLOW_REGISTER
        # system_features.is_allow_create_workspace = dify_config.ALLOW_CREATE_WORKSPACE
        # 修改
        system_features.is_allow_create_workspace = True
        system_features.is_email_setup = dify_config.MAIL_TYPE is not None and dify_config.MAIL_TYPE != ""
...
```

`account_service.py`：

```
class AccountService:
...
@staticmethod
    def load_user(user_id: str) -> None | Account:
        account = db.session.query(Account).filter_by(id=user_id).first()
        if not account:
            return None

        if account.status == AccountStatus.BANNED.value:
            raise Unauthorized("Account is banned.")

        current_tenant = db.session.query(TenantAccountJoin).filter_by(account_id=account.id, current=True).first()
        if current_tenant:
            account.set_tenant_id(current_tenant.tenant_id)
        else:
            available_ta = (
                db.session.query(TenantAccountJoin)
                .filter_by(account_id=account.id)
                # 新增
                .filter_by(role="owner")
                .order_by(TenantAccountJoin.id.asc())
                .first()
            )
            if not available_ta:
                return None

            account.set_tenant_id(available_ta.tenant_id)
            available_ta.current = True
            db.session.commit()

        if datetime.now(UTC).replace(tzinfo=None) - account.last_active_at > timedelta(minutes=10):
            account.last_active_at = datetime.now(UTC).replace(tzinfo=None)
            db.session.commit()

        return cast(Account, account)
...
```

然后，把`feature_service.py`和`account_service.py`分别上传到`api`和`worker`容器：

```
docker container cp account_service.py dify-api-1:/app/api/services/account_service.py
docker container cp feature_service.py dify-api-1:/app/api/services/feature_service.py
docker container cp account_service.py dify-worker-1:/app/api/services/account_service.py
docker container cp feature_service.py dify-worker-1:/app/api/services/feature_service.py
```

最后，重启`api`和`worker`容器。

之后，再在**设置 −> 成员**功能中，添加新的团队成员。新的团队成员登录后，就能看到2个工作空间：**被邀请加入**的和**自己**的。

