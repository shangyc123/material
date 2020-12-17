在项目的.idea文件夹的workspace.xml里

找到component为RunDashboard的节点

在其中加入：

```xml
<option name="configurationTypes">
      <set>
        <option value="SpringBootApplicationConfigurationType" />
      </set>
</option>
```

