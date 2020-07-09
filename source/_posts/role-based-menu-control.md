---
title: 一种基于RBAC模型的菜单访问权限控制
date: 2020/7/9 16:00:00
---


## 场景

在许多项目中，针对不同角色的用户显示不同的菜单，从而进行权限控制，是一个很常见的场景。例如管理员可以访问网站的后台管理，而普通用户只可以访问其拥有权限的页面。有诸多框架，如shiro,spring security 都可以用于解决这一问题。当项目规模较小、逻辑比较简单时，我们没有必要使用这些重量级的框架。这里提供一个工具类，可以快速实现针对不同角色的用户，显示相对应的菜单这一功能。

## 思路

1. 设计一颗树，包含全量的菜单，每个节点存储菜单的名称、地址信息。
2.  在数据库中建立用户ID和角色的对应关系。
3. 初始化不同角色拥有的菜单权限。
4. 当用户登录时，首先查询其角色，根据角色所拥有的菜单节点，调整步骤一中的菜单树，返回给前端。

## 实现

### 设计一个菜单节点MenuNode

```java
import lombok.Data;
import java.util.List;

@Data
public class MenuNode {
    private String title;
    private String url;
    private List<MenuNode> child;
}

```

### 将全量的菜单组织成一个json文件,这里将其放在resource目录下

```json
[
  {
    "title": "首页",
    "url": "home"
  },

  {
    "title": "文章",
    "url": "article"
  },

  {
    "title": "后台",
    "url": "admin",
    "child": [
      {
        "title": "商品管理",
        "url": "product"
      },
      {
        "title": "用户管理",
        "url": "user"
      }
    ]
  }
]
```

### 建立一个角色类，将角色与拥有的菜单进行关联

```java
@Data
public class RoleInfo {

    /**
     * 角色编号
     */
    private String roleNumber;

    /**
     * 角色名
     */
    private String roleName;
    /**
     * 菜单中文名
     */
    private List<String> menuPermission;

    public RoleInfo() {
    }

    public RoleInfo(String roleNumber, String roleName, List<String> menuPermission) {
        this.roleNumber = roleNumber;
        this.roleName = roleName;
        this.menuPermission = menuPermission;
    }
}

```

### 通过下面的工具类，获取用户对应的菜单

```java
package io.mhcn.github.menuauth.utils;
import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.io.FileUtils;
import java.io.File;
import java.io.IOException;
import java.util.*;

@Slf4j
public class MenuUtils {

    private static final RoleInfo SYS_ADMIN =
            new RoleInfo("01","超级管理员", Arrays.asList("首页","商品管理","后台","用户管理"));
    private static final RoleInfo OPERATION =
            new RoleInfo("02","运营", Arrays.asList("首页","商品管理","文章"));
    private static final RoleInfo USER =
            new RoleInfo("03","用户", Arrays.asList("首页","文章"));

    private static final List<RoleInfo> wholeRoleList = Arrays.asList(SYS_ADMIN,OPERATION,USER);

    private static String allMenuStr;

    static {
        try {
            log.info("menu classpath root "+ MenuUtils.class.getResource("/"));
            log.info("json file path " + MenuUtils.class.getResource("/MENU.json"));
            log.info("getPath output " + MenuUtils.class.getResource("/MENU.json").getPath());
            allMenuStr = FileUtils.readFileToString(new File(MenuUtils.class.getResource("/MENU.json").getPath()),"UTF-8");
        }catch (IOException e){

        }
    }

    /**
     * 获取权限菜单
     * @param roleNumSet
     * @return
     */
    public static List<MenuNode> getMenuList(Set<String> roleNumSet){
        List<RoleInfo> roleInfoList = new ArrayList<>();

        for (RoleInfo role : wholeRoleList){
            if (roleNumSet.contains(role.getRoleNumber())){
                roleInfoList.add(role);
            }
        }

        Set<String> menu = new HashSet<>();
        for (RoleInfo role : roleInfoList){
            menu.addAll(role.getMenuPermission());
        }

        List<MenuNode> allMenuList = JSON.parseArray(allMenuStr, MenuNode.class);

        //从全量的菜单中过滤掉没有权限的部分
        removeNotExist(allMenuList,menu);
        return allMenuList;
    }


    /**
     * 移除没有权限的菜单
     * @param menuNodeList 全部的菜单
     * @param menus  登录用户有权限的菜单
     */
    private static void removeNotExist(List<MenuNode> menuNodeList, Set<String> menus){
        if (CollectionUtils.isEmpty(menuNodeList)) {
            return;
        }
        Iterator<MenuNode> iterator = menuNodeList.iterator();
        while (iterator.hasNext()){
            MenuNode menuNode = iterator.next();
            if (CollectionUtils.isNotEmpty(menuNode.getChild())){
                removeNotExist(menuNode.getChild(), menus);
            }
            if (!menus.contains(menuNode.getTitle()) && CollectionUtils.isEmpty(menuNode.getChild())){
                iterator.remove();
            }
        }
    }
}

```

### 测试

准备工作完成了，写一个controller测试一下

```java
@RestController
public class TestingController {
    @GetMapping("/getTree")
    public List<MenuNode> getMenuTree(){
        Set<String> roleSet = new HashSet<>();
        roleSet.add("01");
        roleSet.add("03");
        List<MenuNode> menuNodeList = MenuUtils.getMenuList(roleSet);
        return menuNodeList;
    }
}
```

结果如图，符合我们的预期

![hexo_post_1.png](https://i.loli.net/2020/07/09/WwjREvzSkyKr1Bc.png)

成功✌️