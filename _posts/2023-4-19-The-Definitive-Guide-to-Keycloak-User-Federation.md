---
layout: post
title: Keycloak User Federation权威指南
---

之前，写过一篇[Keycloak User Federation入门](https://warnier-zhang.github.io/Introduction-to-Keycloak-User-Federation/)，简单介绍了Keycloak User Storage SPI的开发、测试、部署方法；今天，就用本篇博客来详细说说如何实现User Storage SPI，它背后的原理、处理逻辑以及疑难点。

**User Storage SPI**中的重要接口如下：

- `org.keycloak.storage.UserStorageProviderFactory`：工厂模式，用于创建`UserStorageProvider`对象；
- `org.keycloak.storage.UserStorageProvider`：
- `org.keycloak.storage.user.UserLookupProvider`：查找用户信息；
- `org.keycloak.storage.user.UserQueryProvider`：查询用户列表，用于在管理员控制台里查看、管理所有用户信息；
- `org.keycloak.storage.user.UserRegistrationProvider`：新增、删除用户
- `org.keycloak.storage.user.UserBulkUpdateProvider`：批量更新用户信息；
- `org.keycloak.credential.CredentialInputValidator`：校验密码；
- `org.keycloak.credential.CredentialInputUpdater`：修改密码；
- `org.keycloak.models.UserModel`：用户信息；

### 用户管理

#### 查找用户信息

Keycloak查找用户的步骤：

1. 缓存；
2. Keycloak本地数据库；
3. 遍历在User Federation中配好的Provider列表，直到匹配到相应的用户信息；

根据用户名、ID、邮件地址查找用户信息需要实现`UserLookupProvider`接口中的`getUserByUsername()`、`getUserById()`、`getUserByEmail()`方法，其中：

- ID，指Keycloak ID，是`StorageId`对象，格式为：`f:[插件ID]:[用户ID]`，插件ID是Keycloak自动生成的，用户ID指外部用户的登录名或ID；

- 上述3个方法的返回值都是`UserModel`对象，其适配器类为`AbstractUserAdapter`和`AbstractUserAdapterFederatedStorage`，适配器类默认把`username`作为`用户ID`来生成Keycloak ID；

  一般情况下，**<u>都是把一个派生于`AbstractUserAdapterFederatedStorage`的子类`UserAdapter`作为返回值</u>**。

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, UserQueryProvider, CredentialInputValidator, CredentialInputUpdater {
	...
	
    @Override
    public UserModel getUserById(String id, RealmModel realm) {
        String username = (new StorageId(id)).getExternalId();
        return getUserByUsername(realm, username);
    }

    @Override
    public UserModel getUserByUsername(String username, RealmModel realm) {
        User user = this.loadUserByUsername(username);
        return user != null ? new UserAdapter(this.session, realm, this.config, user) : null;
    }

    private User loadUserByUsername(String username) {
        ...
    }

    @Override
    public UserModel getUserByEmail(String email, RealmModel realm) {
        return null;
    }
    
    ...
}
```

大家可以自行用`JDBC`或者**`Apache Commons DbUtils`**实现具体的查询外部数据库的代码！

##### 自定义用户属性（Attribute）

除了登录名（`username`）、姓名（`firstName`、`lastName`）、邮件地址（`email`）这些之外，Keycloak支持给用户添加任意的属性。每个属性名可以有多个值。

可以通过`UserModel`中的`getAttributes()`方法实现：

```
@Override
public Map<String, List<String>> getAttributes() {
    MultivaluedHashMap<String, String> attributes = new MultivaluedHashMap<String, String>();
    attributes.add(UserModel.USERNAME, this.user.getUsername());
    attributes.add(UserModel.FIRST_NAME, this.user.getFirstName());
    attributes.add(UserModel.LAST_NAME, this.user.getLastName());
    attributes.add(UserModel.EMAIL, this.user.getEmail());
    if (StringUtils.isNotEmpty(this.user.getRoles())) {
		attributes.add("roles", this.user.getRoles());
	}
	if (StringUtils.isNotEmpty(this.user.getAuthorities())) {
		attributes.add("authorities", this.user.getAuthorities());
	}
	if (StringUtils.isNotEmpty(this.user.getResources())) {
		attributes.add("resources", this.user.getResources());
	}
	return attributes;
}
```

#### 校验密码

验证用户输入的密码是否正确，需要实现`CredentialInputValidator`接口中的`isValid()`方法，此处可以利用上述查找用户信息的部分实现。

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, UserQueryProvider, CredentialInputValidator, CredentialInputUpdater {
	...
	
    public boolean supportsCredentialType(String credentialType) {
        return PasswordCredentialModel.TYPE.equals(credentialType);
    }

    public boolean isConfiguredFor(RealmModel realm, UserModel user, String credentialType) {
        return this.supportsCredentialType(credentialType);
    }

    public boolean isValid(RealmModel realm, UserModel user, CredentialInput credentialInput) {
        if (!supportsCredentialType(credentialInput.getType())) {
            return false;
        }

        if (StringUtils.isEmpty(credentialInput.getChallengeResponse())) {
            return false;
        }

        // 检查用户是否存在；
        User externalUser = this.loadUserByUsername(user.getUsername());
        if (externalUser == null || StringUtils.isEmpty(externalUser.getPassword())) {
            return false;
        }

        // 检查密码是否正确；
        return externalUser.getPassword().equals(this.encodePassword(credentialInput.getChallengeResponse()));
    }

    private String encodePassword(String password) {
        ...
    }
    
    ...
}
```

#### 修改密码

修改密码需要实现`CredentialInputUpdater`接口中的`updateCredential()`方法，注意点是在修改完成之后，**<u>需要刷新Keycloak中的用户信息（UserAdaper）</u>**，否则会造成用户信息不一致。

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, UserQueryProvider, CredentialInputValidator, CredentialInputUpdater {
	...

    @Override
    public boolean updateCredential(RealmModel realm, UserModel user, CredentialInput credentialInput) {
        ...
        
        String password = credentialInput.getChallengeResponse();
        PolicyError error = this.session.getProvider(PasswordPolicyManagerProvider.class).validate(realm, user, password);
        if (error != null) {
            throw new ModelException(error.getMessage(), error.getParameters());
        }
		...
    }

    @Override
    public void disableCredentialType(RealmModel realm, UserModel user, String credentialType) {

    }

    @Override
    public Set<String> getDisableableCredentialTypes(RealmModel realm, UserModel user) {
        return Collections.EMPTY_SET;
    }
    
    ...
}
```

可参考上述代码获得`PasswordPolicyManagerProvider`对象来验证用户输入的新密码是否符合要求。

##### 首次登录之后强制修改初始密码

Required Actions是用户在<u>首次登录之后</u>、<u>进入首页之前</u>必须完成的任务、步骤，再次登录无需操作。

默认的Required Actions有：

- Update Password：修改密码；

- Configure OTP：双重认证、动态口令；

- Verify Email：邮箱验证；

- Update Profile：补充个人信息；
- Terms and Conditions：协议、隐私政策等，对应`terms.ftl`；

如果需要用户在首次登录之后修改初始密码，或者在密码失效之后提供新密码，可以通过`UserModel`中的`getRequiredActions()`方法实现：

```
@Override
public Set<String> getRequiredActions() {
	Date now = new Date();
    Set<String> requiredActions = super.getRequiredActions();
    if (this.user.getPasswordChangedTime() == null || (this.user.getPasswordExpiredTime() != null && now.after(this.user.getPasswordExpiredTime()))) {
    if (requiredActions == null) {
    	requiredActions = new HashSet<>();
    }
    	requiredActions.add(RequiredAction.UPDATE_PASSWORD.toString());
    }
    return requiredActions;
}
```

除此之外，还需要配合上述`CredentialInputUpdater`接口来更新外部数据库用户表中的密码。

#### 查询用户列表

![用户管理](..\images\2023\4\19\1.png)

在**Manage > Users**功能中会用到`UserQueryProvider`和`UserRegistrationProvider`接口，`UserQueryProvider`用来查询、检索用户信息，`UserRegistrationProvider`用来新增、删除用户。

查询条件输入的内容可能是用户名、姓名、邮件地址等等。查询到的用户列表，既包含Keycloak本地数据库中的用户，又包含provider提供的外部数据库中的用户。

在`UserQueryProvider`接口中，最重要方法就是`getUsersCount(RealmModel realm, String search)` 、`getUsers(RealmModel realm, int firstResult, int maxResults)`和`searchForUser(Map<String, String> params, RealmModel realm, int firstResult, int maxResults)` 等3个方法，其他方法均可通过不同传参调用到这3个方法：

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, UserQueryProvider, CredentialInputValidator, CredentialInputUpdater {
	...

    @Override
    public int getUsersCount(RealmModel realm) {
        return this.getUsersCount(realm, "%");
    }

    @Override
    public int getUsersCount(RealmModel realm, Set<String> groupIds) {
        return this.getUsersCount(realm, "%");
    }

    @Override
    public int getUsersCount(RealmModel realm, String search) {
        ...
    }

    @Override
    public List<UserModel> getUsers(RealmModel realm) {
        return this.getUsers(realm, 0, Integer.MAX_VALUE);
    }

    @Override
    public List<UserModel> getUsers(RealmModel realm, int firstResult, int maxResults) {
        ...
    }

    @Override
    public List<UserModel> searchForUser(String search, RealmModel realm) {
        return this.searchForUser(search, realm, 0, Integer.MAX_VALUE);
    }

    @Override
    public List<UserModel> searchForUser(String search, RealmModel realm, int firstResult, int maxResults) {
        Map<String, String> params = new HashMap<>();
        params.put(UserModel.USERNAME, search);
        params.put(UserModel.FIRST_NAME, search);
        params.put(UserModel.LAST_NAME, search);
        params.put(UserModel.EMAIL, search);
        return this.searchForUser(params, realm, firstResult, maxResults);
    }

    @Override
    public List<UserModel> searchForUser(Map<String, String> params, RealmModel realm) {
        return this.searchForUser(params, realm, 0, Integer.MAX_VALUE);
    }

    @Override
    public List<UserModel> searchForUser(Map<String, String> params, RealmModel realm, int firstResult, int maxResults) {
        if (params == null) {
            params = new HashMap<>();
            params.put(UserModel.USERNAME, "%");
            params.put(UserModel.FIRST_NAME, "%");
            params.put(UserModel.LAST_NAME, "%");
            params.put(UserModel.EMAIL, "%");
        }
        ...
    }

    @Override
    public List<UserModel> getGroupMembers(RealmModel realm, GroupModel group) {
        return Collections.EMPTY_LIST;
    }

    @Override
    public List<UserModel> getGroupMembers(RealmModel realm, GroupModel group, int firstResult, int maxResults) {
        return Collections.EMPTY_LIST;
    }

    @Override
    public List<UserModel> searchForUserByUserAttribute(String attrName, String attrValue, RealmModel realm) {
        return Collections.EMPTY_LIST;
    }
    
    ...
}
```



#### 新增用户

#### 删除用户 

### 配置Provider

外部数据库中的连接串、表结构、用户密码的加密方式等等都是不确定的，如果在代码中写死，就会失去灵活性，好在Keycloak支持动态配置重写`UserStorageProviderFactory`中的`getConfigProperties()`方法即可，`getConfigProperties()`返回所有Provider配置项；

主要有2种类型的Provider配置项，一种是文本：

```
ProviderConfigurationBuilder
.create()
.property().name("jdbcUrl").type(ProviderConfigProperty.STRING_TYPE).label("JDBCURL").defaultValue("")
.add()
.build();
```

另一种是下拉列表：

```
ProviderConfigurationBuilder          
.create()
.property().name("passwordEncoder").type(ProviderConfigProperty.LIST_TYPE).label("Password Encoder").defaultValue("MD5").options(Arrays.asList("MD5", "SHA1"))
.add()
.build();
```

### Token令牌

#### 自定义JWT Claims

![用户管理](..\images\2023\4\19\2.png)

可以在**Clients > [XXX Client] > Mappers**功能中新增生成的JWT Token的Claims；如果想把用户信息中的某个`Attribute`添加到Token中，就会用到`UserAttributeMapper`类，背后会调用`UserModel`中的`getAttribute()`方法来获得相应的属性值。

如果该`Attribute`是由某个Provider提供的，就要在`UserAdapter`中重写`getAttribute()`方法。

```
@Override
public List<String> getAttribute(String name) {
    List<String> values = new ArrayList<>();
    if ("roles".equals(name)) {
        if (StringUtils.isNotEmpty(this.user.getRoles())) {
        	values.add(this.user.getRoles());
        }
    } else if ("resources".equals(name)) {
        if (StringUtils.isNotEmpty(this.user.getResources())) {
        	values.add(this.user.getResources());
        }
    } else {
    	values = super.getAttribute(name);
    }
    return values;
}
```

