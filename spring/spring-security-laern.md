# Spring Security

## 文档相关:

{% embed url="https://spring.io/projects/spring-security\#learn" %}

## 1.SpringBoot 整合 Security

### 1.add Depending

```markup
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 2.Config

```java
package com.ml.jkeep.internal.auth;

import com.alibaba.fastjson.JSONObject;
import com.ml.jkeep.common.constant.Common;
import com.ml.jkeep.common.constant.ResultMsg;
import com.ml.jkeep.common.vo.RestVo;
import com.ml.jkeep.service.system.impl.AuthServiceImpl;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.*;
import org.springframework.security.config.annotation.ObjectPostProcessor;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.web.access.intercept.FilterSecurityInterceptor;

import javax.servlet.http.HttpServletResponse;
import java.io.PrintWriter;

/**
 * 安全认证配置
 *
 * @author 谭良忠
 * @date 2019/6/20 10:27
 */
@Slf4j
@EnableWebSecurity
public class JKeepSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private AuthServiceImpl userAuthService;
    @Autowired
    private JKeepSecurityMetadataSource securityMetadataSource;
    @Autowired
    private JKeepAccessDecisionManager accessDecisionManager;
    @Autowired
    private JKeepAccessDeniedHandler accessDeniedHandler;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userAuthService)    // 身份认证
                .passwordEncoder(new BCryptPasswordEncoder()); // 加密方式
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                        o.setSecurityMetadataSource(securityMetadataSource); // 过滤器调用安全元数据源
                        o.setAccessDecisionManager(accessDecisionManager);    // 鉴权策略
                        return o;
                    }
                })
                .and()
                .formLogin()
                .loginPage(Common.LOGIN_PAGE_URL)
                .loginProcessingUrl("/login")
                .usernameParameter("username").passwordParameter("password")
                .failureHandler((req, resp, e) -> {
                    log.error("登陆失败处理 : {}", JSONObject.toJSONString(req.getParameterMap()));
                    ResultMsg resultMsg;
                    if (e instanceof BadCredentialsException ||
                            e instanceof UsernameNotFoundException) {
                        resultMsg = ResultMsg.LOGIN_FAIL_WRONG_PASSWORD;
                    } else if (e instanceof LockedException) {
                        resultMsg = ResultMsg.LOGIN_FAIL_LOCKED;
                    } else if (e instanceof CredentialsExpiredException) {
                        resultMsg = ResultMsg.LOGIN_FAIL_CREDENTIALS_EXPIRED;
                    } else if (e instanceof AccountExpiredException) {
                        resultMsg = ResultMsg.LOGIN_FAIL_ACCOUNT_EXPIRED;
                    } else if (e instanceof DisabledException) {
                        resultMsg = ResultMsg.LOGIN_FAIL_DISABLED;
                    } else {
                        resultMsg = ResultMsg.LOGIN_FAIL;
                    }
                    if (req.getHeader(Common.CSRF_TOKEN_KEY) != null) {
                        // Ajax 提交
                        resp.setStatus(HttpServletResponse.SC_OK);
                        resp.setContentType("application/json;charset=UTF-8");
                        PrintWriter out = resp.getWriter();
                        out.write(JSONObject.toJSONString(RestVo.FAIL(resultMsg)));
                        out.flush();
                        out.close();
                    } else {
                        // 表单提交
                        resp.sendRedirect("/login.html");
                    }
                })
                .successHandler((req, resp, auth) -> {
                    if (req.getHeader(Common.CSRF_TOKEN_KEY) != null) {
                        // Ajax 提交
                        resp.setStatus(HttpServletResponse.SC_OK);
                        resp.setContentType("application/json;charset=UTF-8");
                        PrintWriter out = resp.getWriter();
                        out.write(JSONObject.toJSONString(RestVo.SUCCESS("登录成功")));
                        out.flush();
                        out.close();
                    } else {
                        // 表单提交
                        resp.sendRedirect("/index");
                    }
                })
                .and()
                .logout()
                .logoutUrl("/logout")
                .logoutSuccessHandler((req, resp, authentication) -> {
                    //  注销成功重定向登录页
                    if (req.getHeader(Common.CSRF_TOKEN_KEY) != null) {
                        // Ajax 提交
                        resp.setStatus(HttpServletResponse.SC_OK);
                        resp.setContentType("application/json;charset=UTF-8");
                        PrintWriter out = resp.getWriter();
                        out.write(JSONObject.toJSONString(RestVo.SUCCESS("登出成功")));
                        out.flush();
                        out.close();
                    } else {
                        // 表单提交
                        resp.sendRedirect("/login.html");
                    }
                })
                .permitAll()
                .and()
                .exceptionHandling()
                .accessDeniedHandler(accessDeniedHandler)
//                .and().csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        ;
    }

    @Override
    public void configure(WebSecurity web) {
        // 忽略请求, 无需鉴权即可访问
        web.ignoring().antMatchers("/plugins/**", "**.js", "/img/**", "**.css", "/favicon.ico", "/401.html", "/404.html");
    }

}

```

```java
package com.ml.jkeep.internal.auth;

import com.alibaba.fastjson.JSON;
import com.ml.jkeep.common.constant.Common;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.AccessDecisionManager;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.authentication.AnonymousAuthenticationToken;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.InsufficientAuthenticationException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.web.FilterInvocation;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 鉴权策略
 *
 * @author 谭良忠
 * @date 2019/6/20 13:24
 */
@Slf4j
@Component
public class JKeepAccessDecisionManager implements AccessDecisionManager {

    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException {
        FilterInvocation filterInvocation = (FilterInvocation) object;
        String requestUrl = filterInvocation.getRequestUrl();
        log.info("requestUrl : {} , configAttributes: {} , authentication: {}", requestUrl, JSON.toJSONString(configAttributes), JSON.toJSONString(authentication));
        if (!antPathMatcher.match(Common.LOGIN_PAGE_URL, requestUrl)) {
            if (authentication instanceof AnonymousAuthenticationToken) {
                throw new BadCredentialsException("未登录");
            }
        } else {
            // 登陆页面无需鉴权可访问
            return;
        }
        // 验证用户是否有权访问当前地址
        // configAttributes 可以访问 url 的角色
        // authentication.getAuthorities() 用户所有角色
        List<GrantedAuthority> isPermission = new ArrayList<>();
        configAttributes.forEach(configAttribute -> isPermission.addAll(authentication.getAuthorities().stream().filter(authority -> authority.getAuthority().equals(configAttribute.getAttribute())).collect(Collectors.toList())));
        if (isPermission.isEmpty()) {
            log.info("用户权限不足, 无法访问!");
            throw new AccessDeniedException("用户权限不足, 无法访问!");
        }
    }

    @Override
    public boolean supports(ConfigAttribute attribute) {
        return false;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return false;
    }
}

```

```java
package com.ml.jkeep.internal.auth;

import com.alibaba.fastjson.JSON;
import com.ml.jkeep.common.constant.Common;
import com.ml.jkeep.jpa.system.vo.HrefPermissionVo;
import com.ml.jkeep.service.system.impl.HrefPermissionService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.access.SecurityConfig;
import org.springframework.security.web.FilterInvocation;
import org.springframework.security.web.access.intercept.FilterInvocationSecurityMetadataSource;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.util.CollectionUtils;

import java.util.Collection;
import java.util.HashSet;
import java.util.Set;

/**
 * 过滤器调用安全元数据源
 *
 * @author 谭良忠
 * @date 2019/6/20 11:48
 */
@Slf4j
@Component
public class JKeepSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {

    @Autowired
    private HrefPermissionService hrefPermissionService;

    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
        FilterInvocation filterInvocation = (FilterInvocation) object;
        String requestUrl = filterInvocation.getRequestUrl();
        Set<HrefPermissionVo> allPer = hrefPermissionService.hrefPermission();
        Set<String> roles = new HashSet<>();
        for (HrefPermissionVo per : allPer) {
            if (antPathMatcher.match(per.getHref(), requestUrl)) {
                roles.add(per.getCode());
            }
        }
        if (!CollectionUtils.isEmpty(roles)) {
            log.info("JKeepSecurityMetadataSource, requestUrl : {}, roles: {}", requestUrl, JSON.toJSONString(roles));
            return SecurityConfig.createList(roles.toArray(new String[0]));
        }
        log.info("JKeepSecurityMetadataSource, requestUrl : {}, roles: {}", requestUrl, Common.ROLE_DEFAULT);
        // 当 url 没有配置权限控制时, 赋予默认角色可访问.即所有用户都可访问
        return SecurityConfig.createList(Common.ROLE_DEFAULT);
    }


    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return false;
    }
}

```

```java
package com.ml.jkeep.internal.auth;

import com.alibaba.fastjson.JSONObject;
import com.ml.jkeep.common.constant.Common;
import com.ml.jkeep.common.constant.ResultMsg;
import com.ml.jkeep.common.vo.RestVo;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

/**
 * 访问被拒处理
 *
 * @author 谭良忠
 * @date 2019/6/20 11:41
 */
@Slf4j
@Component
public class JKeepAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException {
        log.info("处理访问被拒 , Parameter: {}, RequestURI: {}, RemoteUser: {}, RemoteAddr: {}, Message: {}",
                JSONObject.toJSONString(request.getParameterMap()), request.getRequestURI(),
                request.getRemoteUser(), request.getRemoteAddr(), accessDeniedException.getMessage());
        if (request.getHeader(Common.CSRF_TOKEN_KEY) != null) {
            // Ajax 提交
            response.setStatus(HttpServletResponse.SC_OK);
            response.setContentType("application/json;charset=UTF-8");
            PrintWriter out = response.getWriter();
            out.write(JSONObject.toJSONString(RestVo.FAIL(ResultMsg.ACCESS_DENIED)));
            out.flush();
            out.close();
        } else {
            // 表单提交 . 重定向401页
            response.sendRedirect("/401.html");
        }
    }
}

```

```java
package com.ml.jkeep.service.system.impl;

import com.ml.jkeep.jpa.system.entity.sys.User;
import com.ml.jkeep.jpa.system.entity.sys.UserAuth;
import com.ml.jkeep.jpa.system.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

/**
 * 鉴权 -impl
 *
 * @author 谭良忠
 * @date 2019/6/20 10:58
 */
@Slf4j
@Service
public class AuthServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private HrefPermissionService hrefPermissionService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        log.info("loadUserByUsername, username: {}", username);
        User user = userRepository.findByUsername(username);
        if (user == null) {
            log.error("用户名不存在, {}", username);
            throw new UsernameNotFoundException("请输入正确的用户名");
        }
        UserAuth userAuth = new UserAuth();
        userAuth.setUsername(user.getUsername());
        userAuth.setPassword(user.getPassword());
        userAuth.setHrefPer(hrefPermissionService.hrefPermission(user.getUserId()));
        return userAuth;
    }

}

```

