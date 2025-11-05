Todo CRUD 機能の ExecPlan を作成してください。
例として、以下のような機能提供します:

```typescript
interface Todo {
  id: number;
  title: string;
  completed: boolean;
  tags: string[];
}

// 使用例
const todoManager = new TodoManager();

// Create
const todo1 = todoManager.create({
  title: "Buy groceries",
  completed: false,
  tags: ["shopping", "urgent"],
});
console.log(todo1); // { id: 1, title: "Buy groceries", completed: false, tags: ["shopping", "urgent"] }

// Read
const allTodos = todoManager.getAll();
console.log(allTodos); // [{ id: 1, ... }]

const singleTodo = todoManager.getById(1);
console.log(singleTodo); // { id: 1, title: "Buy groceries", ... }

// Update
const updated = todoManager.update(1, {
  completed: true,
});
console.log(updated); // { id: 1, title: "Buy groceries", completed: true, tags: ["shopping", "urgent"] }

// Delete
const deleted = todoManager.delete(1);
console.log(deleted); // true
console.log(todoManager.getAll()); // []

// フィルター機能
const filtered = todoManager.filterByTag("urgent");
const completed = todoManager.getCompleted();
const pending = todoManager.getPending();
```

TodoManager クラスを実装し、内部ストレージとして Map<number, Todo> を使用してメモリ効率とルックアップ性能を最適化してください。ID 生成には自動採番カウンターを使用し、削除後も採番が巻き戻らないように管理してください。update メソッドでは Partial<Omit<Todo, 'id'>> を受け取り、既存のプロパティとマージする際に、tags 配列は完全置き換えとし、部分的なマージは行わないでください。不変性を保つため、すべてのメソッドで内部 Map を直接変更する際も返り値としてはコピーを返すようにしてください。filterByTag では完全一致ではなく、tags 配列内に指定されたタグが含まれるかをチェックし、getCompleted と getPending では completed フラグで適切にフィルタリングしてください。エッジケースとして、空の todo リスト、存在しない ID 操作、空文字列のタイトル、空配列のタグ、複数のタグを持つ Todo のフィルタリングなどをテストケースに含め、実行可能な完全なコードとして提供してください。
