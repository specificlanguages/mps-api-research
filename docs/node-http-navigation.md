# HTTP node links and explicit project navigation

This note answers how HTTP node links select ports and projects, and how a plugin can open a node in a specific project.
Findings are source-verified for MPS 2024.1, 2025.1, and 2026.1 at the revisions below. No runtime probe or binary
compatibility test was performed.

## Listener and URL generation

`MPSInternalPortManager` extends `org.jetbrains.io.CustomPortServerManagerBase`. Its `getPort()` reads the registry key
`ide.httpsupport.internalPort`, whose default is 63320. `isAvailableExternally()` returns false.
`cannotBind(Exception, int)` emits a warning saying opening nodes on HTTP requests is disabled. `getCurrentPort()`
returns the configured registry value, not a successfully bound port.

The implementation is identical in the inspected 2024.1 and 2025.1 revisions; 2026.1 changes only the generated
annotation. The plugin descriptor registers the manager as `org.jetbrains.customPortServerManager`.

The 2024.1 checkout's bundled platform sources, `lib/src/platform-sources.zip`, establish that
`org.jetbrains.io.SubServer.bind` binds a custom loopback listener, catches bind failures, invokes `cannotBind`, and
returns false. `BuiltInServerManagerImpl` starts the main built-in server before binding custom ports. It is therefore
inferred that occupation of 63320 prevents this custom listener rather than stopping the IDE or all server listeners.
This startup behavior was not runtime-tested, and the platform implementations for later versions were not compared.

Both `NodeReferenceURL_Action` and `HttpSupportUtil.getURL(SNodeReference)` construct URLs using
`MPSInternalPortManager.getCurrentPort()`. A configured port change therefore changes newly generated URLs; failure to
bind does not itself change their port. `HttpSupportUtil.getURL` uses `QueryStringEncoder` and serializes `ref` with
`PersistenceFacade.getInstance().asString(nodeRef)`.

Sources in `plugins/mps-httpsupport/`:

- `META-INF/plugin.xml`, registry keys and custom port manager registrations.
- `source_gen/jetbrains/mps/ide/httpsupport/manager/plugin/MPSInternalPortManager.java`, methods above.
- `solutions/nodeaccess/source_gen/jetbrains/mps/ide/httpsupport/nodeaccess/plugin/NodeReferenceURL_Action.java`.
- `solutions/runtime/source_gen/jetbrains/mps/ide/httpsupport/runtime/base/HttpSupportUtil.java`, `getURL` overloads.

## Routing and project ownership

The HTTP node handler parses the decoded `ref` with `PersistenceFacade.getInstance().createNodeReference(String)`. In
2024.1 and 2025.1 it selects a project by optional `project` name, otherwise the first opened project.
`HttpSupportUtil.getProjectByName` falls back to the first project when the name is unknown. Project names therefore do
not provide a unique or strict routing identity.

In 2026.1 `NodeOpener_RequestHandler.handle()` calls `HandlerUtil.getProjectForNode(Project, SNodeReference)`. This
checks whether the proposed project owns the model, then returns the first owning open project. Ownership is tested
under `computeReadAction`, through `p.getProjectModules()` and `module.getModel(modelId)`. This selection is local to
one MPS process; there is no cross-process chooser.

Successful resolution in a project repository does not establish project ownership. `ProjectRepository` explicitly
exposes the modules of the whole MPS instance; its `getModule`, `getModules`, and `getModel` delegate to the root
repository. A shared library node can be resolvable without its model belonging to any project's own modules.

Evidence:

- `plugins/mps-httpsupport/solutions/nodeaccess/source_gen/jetbrains/mps/ide/httpsupport/nodeaccess/plugin/NodeOpener_RequestHandler.java`,
  parsing and `handle`.
- `plugins/mps-httpsupport/solutions/runtime/source_gen/jetbrains/mps/ide/httpsupport/runtime/base/HttpSupportUtil.java`,
  `getProjectByName` and `getSomeProject`.
- `plugins/mps-httpsupport/solutions/nodeaccess/source_gen/jetbrains/mps/ide/httpsupport/nodeaccess/plugin/HandlerUtil.java`,
  2026.1 lines 66–96, `getProjectForNode` and `ownsModel`.
- `core/project/source/jetbrains/mps/project/ProjectRepository.java`, lines 35–47 and 77–90.

## Explicit navigation and window focus

`jetbrains.mps.openapi.navigation.EditorNavigator` is identical in the three inspected revisions. Its public API
includes `EditorNavigator(Project)`, `shallFocus(boolean)`, `shallSelect(boolean)`,
`onceEditorReady(BiConsumer<SNode, Editor>)`, and `void open(SNodeReference)`.

```java
new EditorNavigator(selectedProject)
    .shallFocus(true)
    .shallSelect(true)
    .open(nodeReference);
```

`open` arranges `selectedProject.getModelAccess().runReadInEDT`, resolves in the selected project's repository, and
passes that same project to `NavigationSupport.openNode`. The caller does not need to arrange EDT/model access for this
call. The editor-ready callback executes inside that EDT/read action only if an editor successfully opens; missing
references produce no callback. `open` returning does not constitute a successful-navigation acknowledgment.

The existing HTTP handler additionally schedules `HandlerUtil.requestFocus(Project)` on the UI thread. That method
asserts EDT and calls `com.intellij.ide.impl.ProjectUtil.focusProjectWindow(ideaProject, true)`, obtaining the IDEA
project from `MPSProject.getProject()`. Actual foreground activation on macOS was not tested.

Evidence:

- `editor/editor-api/source/jetbrains/mps/openapi/navigation/EditorNavigator.java`, lines 59–103.
- `workbench/mps-workbench/source/jetbrains/mps/ide/navigation/NavigationSupportImpl.java`, lines 52–55.
- `workbench/mps-editor/source/jetbrains/mps/ide/editor/MPSEditorOpener.java`, lines 73–75 and 141.
- `plugins/mps-httpsupport/solutions/nodeaccess/source_gen/jetbrains/mps/ide/httpsupport/nodeaccess/plugin/HandlerUtil.java`,
  `openNode` and `requestFocus`.
- `workbench/mps-platform/source/jetbrains/mps/project/MPSProject.java`, `getProject`.

## Versions and unresolved work

| Release | JetBrains/MPS source revision              |
| ------- | ------------------------------------------ |
| 2024.1  | `6236c4073eac3cde78506add6b0fa90601d76009` |
| 2025.1  | `f4d90532bcac5e0339b3161cec38abf49567cffb` |
| 2026.1  | `499c4a0fe6bfe7f95462c33966c72192158446ca` |

All source paths above are repository-relative. HTTP handler and manager classes are in the HTTP Support Plugin;
`EditorNavigator` is in the editor API component (`mps-editor-api.jar`). Source stability does not prove one plugin
binary works on all three releases. Startup with an occupied port, binary compatibility, project disposal races, and
macOS window activation require focused runtime checks.
