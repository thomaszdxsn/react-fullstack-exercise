# react-fullstack-exercise

> [origin repository](https://github.com/ThaddeusJiang/fullstack-assignment)

> [issue](https://github.com/jifa-org/public_space/issues/5)

## 需求分析

> 提供一个申请流程，让用户可以申请变更个人数据，申请被通过时自动更新数据。

1. 三种用户角色，它们的权限不一样，可使用的功能也不一样
2. 三种数实体：Form, DataSource, Flow。Form 和 Flow 都由用户自定义，DataSource 不明？
3. Form 填写后需要和 DataSource 绑定，在 Flow 通过后 DataSource 需要进行更新
4. 需要有操作历史

### 需求评审

- 用户可申请变更「个人数据」，这个个人数据是指后面指代的 DataSource？DataSource 是一个更抽象的概念，无妨直接使用它，也许之后功能可以扩展更多的场景。
- Form 和 DataSource 的绑定
  - 最简单的方式是一对一的绑定，每个 DataSource 的字段代表 Form 的一个字段
  - 如果有一对多/多对一的绑定，肯定还要牵涉到一些逻辑计算，需要定义一套 DSL 的逻辑来实现(like excel formula?)
- Form 定义完成后是可以修改定义的，那么如果定义完成后到修改的这段时间内用户已经提交了申请，怎么办？
  - 用户提交的申请应该复制一套当时的 Form 定义，申请执行过程中/执行完毕都不会受到 Form 定义的影响
  - 如果管理员觉得申请使用的这套 Form 定义已经过时，可选择拒绝
- Flow 的定义
  - 可以生成多级审批 flow （无分叉）
  - 审批可以选择「打回」或者「拒绝」，可以在定义中选择让哪个阶审批人拥有「拒绝」的权利
  - 「打回」的申请可以编辑然后重新提交等待审批

## 架构设计

### 后端

- 定义角色，为角色定义权限
- Flow 在后端的表设计中，不应该是修改状态后然后记录历史，而是直接生成操作历史，然后根据历史来生成状态

> 可以参考复式记账本的逻辑：一个账号的余额由计算所有的转账记录后得出

```
1. {stage: "member submit", "user": "member1", "comment": "...", data: ""(JSON), meta: <form rules> "op": "submit"}
2. {stage: "manager approve", "user": "manager1", "comment": "...", "op": "approve"}
3. {stage: "manager approve", "user": "manager2", "comment": "...", "op": "return"}
// use can update meta & edit data
4. {stage: "member submit", "user": "member1", "comment": "...", data: ""(JSON), meta: <form rules> "op": "submit"}
...
```

执行第 3 步的时候，控制权交还给 `member1`，他可以选择修改后重新 submit，也可以 cancel 这次申请。

Flow 的状态：

1. Draft
2. Need Approve
3. Returned
4. Approved
5. Rejected
6. Cancelled

所有这些状态都不用记录，而是根据上面的操作记录按照时间顺序而得出结果

### 前端

- Form Builder
- Form Diff Viewer
- Stepper
- History

## 数据表设计

```
erDiagram
	MetaForm {
		id	int
		name	string
	}
	
	MetaFormField {
		id		int
		metaFormId	int
		label		string
		type		enum
		required	boolean
		min?		int
		max?		int
	}
	
	MetaFlow {
		id		int
		metaFormId	int
		returnable	boolean
		rejectable	boolean
		index		int
		byType		enum
		byId		id
	}
	
	Form {
		id		int
		metaFormId	int
		registerId	int
	}
	
	FormAction {
		id		int
		formId		int
		flowId		int
		op		enum
		createdAt 	datetime
		data?		json
		meta?		json
		comment?  	text
		file?		blob
	}
```

