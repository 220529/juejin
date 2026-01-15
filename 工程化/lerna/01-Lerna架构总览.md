# Lerna 源码解析：架构总览

> 本文深入分析 Lerna 的核心架构，理解 Command 基类和命令执行流程。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Lerna CLI                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                        yargs                               │  │
│  │   lerna version | publish | run | exec | ...              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Command Factory                         │  │
│  │   new VersionCommand(argv) | new PublishCommand(argv)     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Command Base                            │  │
│  │   configureProperties → initialize → execute              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │   Project   │     │   Package   │     │ ProjectGraph│       │
│  │  项目管理    │     │   包管理    │     │  依赖图     │       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

## CLI 入口

```typescript
// packages/lerna/src/cli.ts
import { lerna } from "@lerna/core";

// 使用 yargs 构建 CLI
const cli = lerna(localCli)
  .command(versionCommand)
  .command(publishCommand)
  .command(runCommand)
  .command(execCommand)
  .command(initCommand)
  .command(listCommand)
  .command(changedCommand)
  .command(diffCommand)
  .command(importCommand)
  .command(cleanCommand)
  .parse(process.argv.slice(2));
```

## Command 基类

所有命令都继承自 Command 基类：

```typescript
// libs/core/src/lib/command/index.ts
export abstract class Command<T extends CommandConfigOptions = CommandConfigOptions> {
  argv: Arguments<T>;
  options: T;
  project: Project;
  projectGraph: ProjectGraphWithPackages;
  projectFileMap: ProjectFileMap;
  logger: Logger;
  concurrency: number;
  toposort: boolean;
  execOpts: ExecOptions;

  constructor(argv: Arguments<T>) {
    this.argv = argv;
    this.options = argv as unknown as T;
  }

  // 模板方法：执行流程
  async runner() {
    // 1. 配置属性
    this.configureProperties();
    
    // 2. 配置环境
    await this.configureEnvironment();
    
    // 3. 配置日志
    this.configureLogging();
    
    // 4. 运行前置检查
    await this.runValidations();
    
    // 5. 运行准备工作
    await this.runPreparations();
    
    // 6. 初始化（子类实现）
    const proceed = await this.initialize();
    
    if (proceed !== false) {
      // 7. 执行（子类实现）
      await this.execute();
    }
  }

  // 子类必须实现
  abstract initialize(): Promise<boolean | void>;
  abstract execute(): Promise<void>;

  // 配置属性
  configureProperties() {
    const { concurrency, sort } = this.options;
    this.concurrency = concurrency || os.cpus().length;
    this.toposort = sort !== false;
  }

  // 配置环境
  async configureEnvironment() {
    // 加载项目配置
    this.project = new Project(this.argv.cwd);
    
    // 构建项目图
    const { projectGraph, projectFileMap } = await this.project.getProjectGraph();
    this.projectGraph = projectGraph;
    this.projectFileMap = projectFileMap;
  }
}
```

## 命令执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      命令执行流程                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     1. configureProperties    │
              │   - 解析并发数                 │
              │   - 配置拓扑排序               │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     2. configureEnvironment   │
              │   - 加载 lerna.json           │
              │   - 构建 ProjectGraph         │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     3. configureLogging       │
              │   - 设置日志级别               │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     4. runValidations         │
              │   - 检查 git 状态              │
              │   - 验证配置                   │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     5. runPreparations        │
              │   - 准备执行环境               │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     6. initialize()           │
              │   - 子类实现                   │
              │   - 返回 false 可中止执行      │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     7. execute()              │
              │   - 子类实现                   │
              │   - 执行核心逻辑               │
              └───────────────────────────────┘
```

## Project 项目管理

```typescript
// libs/core/src/lib/project/index.ts
export class Project {
  rootPath: string;
  manifest: Package;
  config: LernaConfig;
  licensePath?: string;

  constructor(cwd?: string) {
    this.rootPath = findRoot(cwd);
    this.manifest = new Package(
      loadJsonFile(path.join(this.rootPath, "package.json")),
      this.rootPath
    );
    this.config = this.loadConfig();
  }

  // 加载 lerna.json 配置
  loadConfig(): LernaConfig {
    const configPath = path.join(this.rootPath, "lerna.json");
    return loadJsonFile(configPath);
  }

  // 获取版本
  get version(): string {
    return this.config.version;
  }

  // 是否独立版本模式
  isIndependent(): boolean {
    return this.version === "independent";
  }

  // 获取包目录 glob
  get packageConfigs(): string[] {
    return this.config.packages || ["packages/*"];
  }

  // 构建项目图
  async getProjectGraph(): Promise<{
    projectGraph: ProjectGraphWithPackages;
    projectFileMap: ProjectFileMap;
  }> {
    // 使用 Nx 构建项目图
    const nxProjectGraph = await createProjectGraphAsync();
    
    // 增强为带 Package 信息的图
    return buildProjectGraphWithPackages(nxProjectGraph, this.rootPath);
  }
}
```

## Package 包管理

```typescript
// libs/core/src/lib/package.ts
export class Package {
  name: string;
  version: string;
  location: string;
  private: boolean;
  manifestLocation: string;
  
  private _contents?: string;
  private _pkg: PackageJson;

  constructor(pkg: PackageJson, location: string) {
    this._pkg = pkg;
    this.name = pkg.name;
    this.version = pkg.version;
    this.location = location;
    this.private = !!pkg.private;
    this.manifestLocation = path.join(location, "package.json");
  }

  // 获取/设置属性
  get(key: string): unknown {
    return this._pkg[key];
  }

  set(key: string, value: unknown): this {
    this._pkg[key] = value;
    return this;
  }

  // 获取发布目录
  get contents(): string {
    return this._contents || this.location;
  }

  set contents(value: string) {
    this._contents = value;
  }

  // 更新本地依赖版本
  updateLocalDependency(
    resolved: NpaResult,
    depVersion: string,
    savePrefix: string
  ): void {
    const depName = resolved.name;
    const depType = resolved.type;
    
    // 更新 dependencies/devDependencies/peerDependencies
    for (const type of ["dependencies", "devDependencies", "peerDependencies"]) {
      if (this._pkg[type]?.[depName]) {
        this._pkg[type][depName] = `${savePrefix}${depVersion}`;
      }
    }
  }

  // 序列化到文件
  async serialize(): Promise<void> {
    await writeJsonFile(this.manifestLocation, this._pkg);
  }

  // 移除 private 字段
  removePrivate(): void {
    delete this._pkg.private;
  }
}
```

## ProjectGraph 依赖图

```typescript
// libs/core/src/lib/project-graph-with-packages.ts
export interface ProjectGraphWithPackages {
  nodes: Record<string, ProjectGraphProjectNodeWithPackage>;
  dependencies: Record<string, ProjectGraphDependency[]>;
  localPackageDependencies: Record<string, LocalPackageDependency[]>;
}

export interface ProjectGraphProjectNodeWithPackage {
  name: string;
  type: "lib" | "app";
  data: {
    root: string;
    targets?: Record<string, unknown>;
  };
  package?: Package;
}

// 构建带 Package 的项目图
export async function buildProjectGraphWithPackages(
  nxProjectGraph: ProjectGraph,
  rootPath: string
): Promise<ProjectGraphWithPackages> {
  const nodes: Record<string, ProjectGraphProjectNodeWithPackage> = {};
  const localPackageDependencies: Record<string, LocalPackageDependency[]> = {};

  // 遍历 Nx 项目图节点
  for (const [name, node] of Object.entries(nxProjectGraph.nodes)) {
    const packageJsonPath = path.join(rootPath, node.data.root, "package.json");
    
    let pkg: Package | undefined;
    if (existsSync(packageJsonPath)) {
      const pkgJson = loadJsonFile(packageJsonPath);
      pkg = new Package(pkgJson, path.join(rootPath, node.data.root));
    }

    nodes[name] = {
      ...node,
      package: pkg,
    };
  }

  // 分析本地依赖
  for (const [name, deps] of Object.entries(nxProjectGraph.dependencies)) {
    localPackageDependencies[name] = deps
      .filter((dep) => nodes[dep.target]?.package)
      .map((dep) => ({
        source: name,
        target: dep.target,
        targetResolvedNpaResult: npa(nodes[dep.target].package!.name),
      }));
  }

  return {
    nodes,
    dependencies: nxProjectGraph.dependencies,
    localPackageDependencies,
  };
}
```

## 拓扑排序执行

```typescript
// libs/core/src/lib/run-projects-topologically.ts
export async function runProjectsTopologically<T>(
  projects: ProjectGraphProjectNodeWithPackage[],
  projectGraph: ProjectGraphWithPackages,
  runner: (node: ProjectGraphProjectNodeWithPackage) => Promise<T>,
  options: { concurrency: number }
): Promise<T[]> {
  const { concurrency } = options;
  
  // 拓扑排序
  const sorted = toposortProjects(projects, projectGraph);
  
  // 并行执行（尊重依赖顺序）
  const results: T[] = [];
  const inProgress = new Set<string>();
  const completed = new Set<string>();
  
  const queue = new PQueue({ concurrency });

  for (const node of sorted) {
    queue.add(async () => {
      // 等待依赖完成
      const deps = projectGraph.localPackageDependencies[node.name] || [];
      await Promise.all(
        deps.map((dep) => waitForCompletion(dep.target, completed))
      );

      inProgress.add(node.name);
      const result = await runner(node);
      results.push(result);
      inProgress.delete(node.name);
      completed.add(node.name);
    });
  }

  await queue.onIdle();
  return results;
}

// 拓扑排序
export function toposortProjects(
  projects: ProjectGraphProjectNodeWithPackage[],
  projectGraph: ProjectGraphWithPackages
): ProjectGraphProjectNodeWithPackage[] {
  const sorted: ProjectGraphProjectNodeWithPackage[] = [];
  const visited = new Set<string>();
  const visiting = new Set<string>();

  function visit(node: ProjectGraphProjectNodeWithPackage) {
    if (visited.has(node.name)) return;
    if (visiting.has(node.name)) {
      throw new Error(`Circular dependency detected: ${node.name}`);
    }

    visiting.add(node.name);

    const deps = projectGraph.localPackageDependencies[node.name] || [];
    for (const dep of deps) {
      const depNode = projectGraph.nodes[dep.target];
      if (depNode && projects.includes(depNode)) {
        visit(depNode);
      }
    }

    visiting.delete(node.name);
    visited.add(node.name);
    sorted.push(node);
  }

  for (const project of projects) {
    visit(project);
  }

  return sorted;
}
```

## 命令注册

```typescript
// libs/commands/version/src/command.ts
export const command = "version [bump]";
export const describe = "Bump version of packages changed since the last release";

export const builder = (yargs: Yargs) => {
  return yargs
    .positional("bump", {
      describe: "Increment version(s) by explicit version or semver keyword",
      type: "string",
    })
    .option("conventional-commits", {
      describe: "Use conventional-changelog to determine version bump",
      type: "boolean",
    })
    .option("message", {
      alias: "m",
      describe: "Use a custom commit message when creating the version commit",
      type: "string",
    });
};

export const handler = (argv: Arguments) => {
  return new VersionCommand(argv);
};
```

## 小结

Lerna 架构的核心设计：

1. **Command 模式**：所有命令继承 Command 基类，统一执行流程
2. **Project 管理**：加载配置、管理包、构建依赖图
3. **Package 抽象**：封装包的读写和依赖更新
4. **ProjectGraph**：基于 Nx 构建依赖图，支持拓扑排序
5. **拓扑执行**：按依赖顺序并行执行任务

---

> 下一篇：版本管理
