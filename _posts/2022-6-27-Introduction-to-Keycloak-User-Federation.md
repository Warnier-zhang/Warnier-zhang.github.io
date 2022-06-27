---
layout: post
title: Keycloak User Federation入门
---

Keycloak提供了一套基于Java SPI机制的User Storage API，用来集成已有的、外部的用户数据源，默认支持LDAP、Active Directory和Kerberos。

简单来说，就是当Keycloak处理用户登录请求时，通过查询外部的User表来判断用户是否存在，密码是否正确。

User Storage SPI中最重要的2个接口如下：

- `UserStorageProviderFactory`

  负责创建`UserStorageProvider`的实例。

- `UserStorageProvider` 

  配合其他接口，如`UserLookupProvider`等，负责查找用户、校验密码、用户管理等等。每一次用户登录请求都会创建一个新的`UserStorageProvider`实例。

其他重要的接口有：

- `UserLookupProvider`

  根据用户名、ID、Email等查找用户信息；

- `CredentialInputValidator`

  校验用户输入的密码；

下面演示如何扩展User Storage SPI来查找存储在MySQL中的用户信息。

## UserStorageProviderFactory

```
public class JdbcStorageProviderFactory implements UserStorageProviderFactory<JdbcStorageProvider> {
    public static final String PROVIDER_NAME = "jdbc";
    
    public JdbcStorageProvider create(KeycloakSession session, ComponentModel config) {
        String url = "";
        String driverClassName = "com.mysql.cj.jdbc.Driver";
        String username = "";
        String password = "";
        Connection connection = null;
        try {
            connection = DBUtils.openConnection(driverClassName, url, username, password);
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return new JdbcStorageProvider(session, config, connection);
    }

    public String getId() {
        return PROVIDER_NAME;
    }
}
```

- `getId()`方法

  返回Provider名称；

- `create()`方法

  创建`UserStorageProvider`实例。

  **参数`config`包含自定义的属性，可通过覆盖`getConfigProperties`方法配置，配好之后，Keycloak会根据它自动生成表单供用户输入；**

## UserStorageProvider

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, CredentialInputValidator {
    private KeycloakSession session;
    private ComponentModel config;
    private Connection connection;

    public JdbcStorageProvider(KeycloakSession session, ComponentModel config, Connection connection) {
        this.session = session;
        this.config = config;
        this.connection = connection;
    }

    public void close() {
        DBUtils.closeConnection(this.connection);
        this.connection = null;
    }
    
    ...
    
}
```

当一次用户登录请求处理完时，会先调用`close()` 方法，然后销毁`UserStorageProvider`实例、进行垃圾回收。如果需要释放资源，就可以在`close()` 方法内处理。

## UserLookupProvider

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, CredentialInputValidator {

    ...
    
    @Override
    public UserModel getUserById(String id, RealmModel realm) {
        StorageId storageId = new StorageId(id);
        String username = storageId.getExternalId();
        return getUserByUsername(realm, username);
    }

    @Override
    public UserModel getUserByUsername(String username, RealmModel realm) {
        PreparedStatement stmt = null;
        ResultSet rs = null;
        try {
            stmt = this.connection.prepareStatement("SELECT * FROM user");
            stmt.setString(1, username);
            rs = stmt.executeQuery();
            return this.createUserAdapter(realm, rs);
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            DBUtils.closeResultSet(rs);
            DBUtils.closeStatement(stmt);
        }
        return null;
    }

    @Override
    public UserModel getUserByEmail(String email, RealmModel realm) {
        return null;
    }

    private UserAdapter createUserAdapter(RealmModel realm, ResultSet rs) throws SQLException {
        UserAdapter userAdapter = null;
        if (rs.next()) {
            User user = User.builder()
                    .id(rs.getInt("id"))
                    .username(rs.getString("username"))
                    .firstName(rs.getString("first_name"))
                    .lastName(rs.getString("last_name"))
                    .email(rs.getString("email"))
                    .roles(rs.getString("roles"))
                    .authorities(rs.getString("authorities")))
                    .resources(rs.getString("resources"))
                    .build();
            userAdapter = new UserAdapter(this.session, realm, this.config, user);
        }
        return userAdapter;
    }
}
```

当用户登录时，会触发`getUserByUsername()`方法，根据用户名查找用户信息，返回结果是`UserModel`接口的实例。

本例中的`UserAdapter`扩展自`UserModel`接口的适配器类`AbstractUserAdapter`。`AbstractUserAdapter`的特点是自动把`username`作为`externalId`来生成`userId`；

当生成Token时，会触发`getUserById()`方法。

## CredentialInputValidator

```
public class JdbcStorageProvider implements UserStorageProvider, UserLookupProvider, CredentialInputValidator {

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

        PreparedStatement stmt = null;
        ResultSet rs = null;
        String password = null;
        try {
            stmt = this.connection.prepareStatement("SELECT * FROM user");
            stmt.setString(1, user.getUsername());
            rs = stmt.executeQuery();
            if (rs.next()) {
                password = rs.getString("password");
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            DBUtils.closeResultSet(rs);
            DBUtils.closeStatement(stmt);
        }
        if (password == null) {
            return false;
        }
        return password.equals(DigestUtils.md5Hex(credentialInput.getChallengeResponse()));
    }
}
```

`isValid()`方法负责校验用户输入的密码。

## 编译、打包、部署

为了让Keycloak能自动识别、加载上述User Storage SPI插件，需要在`classpath`中添加`META-INF/services/org.keycloak.storage.UserStorageProviderFactory`文件，文件内容是`JdbcStorageProviderFactory`类的完整路径，即：

```
org.keycloak.storage.jdbc.JdbcStorageProviderFactory
```

然后，执行`mvn clean package`命令打包插件，注意点是**一定要把依赖的第三方库文件一起打包进去**。

最后，把打包好的jar文件拷贝到Keycloak目录中。如果Keycloak是用Docker部署的，就直接执行如下的命令：

```
docker container cp keycloak-jdbc-federation-1.0.0-jar-with-dependencies.jar 容器ID:/opt/jboss/keycloak/standalone/deployments/
```

