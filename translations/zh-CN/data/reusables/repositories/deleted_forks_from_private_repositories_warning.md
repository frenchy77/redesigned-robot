{% warning %}

**警告：**

- 如果您删除某人访问私有仓库的权限，则其对该私有仓库的任何复刻也会被删除。 将保留私人仓库的本地克隆。 如果撤销团队对私有仓库的访问权限，或者删除对私有仓库具有访问权限的团队，并且团队成员无法通过另一个团队访问仓库，则该仓库的私有复刻将被删除。{% ifversion ghes %}
- 当 [LDAP 同步启用](/enterprise/{{ page.version }}/admin/guides/user-management/using-ldap/#enabling-ldap-sync)后，如果从仓库删除某用户，此用户将失去访问权，但其复刻不会被删除。 如果此用户在三个月内被加入具有原组织仓库访问权限的团队，则其对复刻的访问权限将在下次同步时自动恢复。{% endif %}
- 您负责确保无法访问仓库的人员删除任何机密信息或知识产权。

- 对私有{% ifversion fpt or ghes or ghae %}或内部{% endif %}仓库拥有管理员权限的人可以禁止对该仓库进行复刻，组织所有者可以禁止对组织中的任何私有{% ifversion fpt or ghes or ghae %}或内部{% endif %}仓库进行复刻。 更多信息请参阅“[管理组织的复刻政策](/organizations/managing-organization-settings/managing-the-forking-policy-for-your-organization)”和“[管理仓库的复刻政策](/github/administering-a-repository/managing-the-forking-policy-for-your-repository)”。

{% endwarning %}
