---
name: winui-wpf-migration
description: "Migrate WPF desktop apps to WinUI 3: project files, XAML, MVVM, resources, windowing, imaging, installers, and tests. Use for WPF ports and migration regressions. Preserve existing behavior and deployment; not for unrelated Windows App SDK work."
---

# WPF to WinUI 3 Migration

Port WPF code to WinUI 3 while preserving the application's behavior, deployment, stored data, and external contracts. Work through project setup, code and resource conversion, installer changes, and validation in dependency order.

## When to Use

- Convert WPF XAML, controls, bindings, or `System.Windows.*` APIs to WinUI 3.
- Port dispatcher calls, windows, dialogs, file pickers, or imaging code.
- Adapt observable properties, commands, resources, and tests during a WPF migration.
- Fix project, runtime, installer, or signing failures caused by that migration.

Use `winui-design` for target-control samples and visual design, and `winui-dev-workflow` for project setup, per-app analyzers, and build/launch verification. Preserve the migration's required behavior and deployment when using those workflows. Use `winui-packaging` when packaging or distribution is in scope.

## Before Editing

Read the affected application, shared libraries, tests, and build/installer files. Identify:

- The target framework, minimum Windows version, Windows App SDK version, and supported architectures.
- Packaged, packaged-with-external-location, or unpackaged deployment, and the separate .NET and Windows App SDK runtime-deployment choices.
- Existing MVVM/DI conventions, resource accessors, window base classes, and package-version management.
- Settings names and values, command-line options, activation identifiers, and IPC/COM contracts that callers already use.
- The repository's permitted build, test, launch, and packaging checks.

Inventory WPF dependencies and markup outside generated output:

```powershell
rg -n -g '!**/bin/**' -g '!**/obj/**' 'UseWPF|Microsoft\.WindowsDesktop\.App\.WPF|Presentation(Core|Framework)|System\.Windows' .
rg -n -g '*.xaml' -g '!**/bin/**' -g '!**/obj/**' 'DynamicResource|x:Static|x:Type|MultiBinding|Style\.Triggers|DataTrigger|EventTrigger|Window\.Resources|clr-namespace:|pack://' .
rg -n -g '*.cs' -g '!**/bin/**' -g '!**/obj/**' 'Dispatcher|Window\.Current|Application\.Current\.MainWindow|MessageBox|File(Open|Save)Picker|FolderPicker|System\.Windows\.Media\.Imaging' .
```

Before implementing replacements, verify APIs against the restored target WinUI project's references. With WinApp CLI 0.7+, run from that app directory:

```powershell
winapp find-api DispatcherQueue --json --project-dir .
winapp find-api members DispatcherQueue --filter TryEnqueue --json --project-dir .
winapp find-api check-property ListView ItemsSource SelectionMode --json --project-dir .
```

Use either `--project-dir <app-directory>` or `--project <name>` for project selection, never both. Use `winapp find-ui` for usage examples. If the lookup tool is unavailable, inspect the target packages and official API documentation and identify any assumptions that remain unverified.

## Migration Sequence

Port bounded groups of files in dependency order:

1. **Project files** — prepare the WinUI target, package references, XAML/resource items, native references, and assets. Keep assembly names and deployment settings required by existing callers.
2. **Models and shared logic** — remove UI-framework dependencies from reusable logic. Share CLI/UI behavior through a library when needed, without executable-to-executable project references.
3. **MVVM** — adapt observable properties and commands. When adopting CommunityToolkit.Mvvm, preserve setter transformations, defaults, serialization attributes, notifications, and command behavior.
4. **Resource strings** — migrate XAML-localized strings to `.resw`, connect the generated PRI resource index and accessor, and keep stable keys.
5. **Services and imaging** — replace WPF types and propagate asynchronous APIs through callers. Preserve codec, metadata, cancellation, and error behavior.
6. **View models** — move UI updates onto their owning dispatcher queue and adapt binding/command notifications.
7. **Views and pages** — start with leaf views. Port markup, styles, converters, templates, keyboard behavior, and accessibility.
8. **Main view and shell** — integrate the converted views, navigation, windows, and owner-dependent dialogs/pickers.
9. **Application startup** — merge initialization, resources, activation, and cleanup into the WinUI lifecycle.
10. **Installer and build pipeline** — update resource/component inventories, output paths, signing inputs, and build/publish dependencies for the resulting artifacts.
11. **Tests** — adapt async calls, resource access, pixel/metadata assertions, settings fixtures, and integration scenarios.

After each group, use the repository-approved checks that exercise the changed behavior. Keep failures attributable to that group.

## Migration Rules

### Projects and application code

- Merge into the target's `App.xaml` and `App.xaml.cs`; preserve generated startup and initialization that the target SDK needs.
- Remove WPF-only dependencies and project items once their callers have been ported. Do not add `UseWPF` or a WPF framework reference to the WinUI application to conceal an incomplete conversion.
- Change `WindowsPackageType`, `WindowsAppSDKSelfContained`, `SelfContained`, or package identity only as part of the chosen deployment configuration.
- Keep reusable logic out of executable project references; check unexpected copied executables, dependency files, and runtime configuration files in the output.
- Follow existing window and MVVM conventions. Use WinUIEx or Toolkit components when the project adopts them for the required behavior.
- Preserve JSON names, enum values, defaults, resource keys, CLI options, and IPC/COM contracts.

### XAML and resources

- Replace `clr-namespace:` with `using:`, and map unsupported markup individually. `x:DataType` enables typed templates but does not reproduce WPF's implicit template selection.
- Use `ThemeResource` for theme-dependent resources, `StaticResource` for fixed resources, and binding/code for other dynamic values.
- Replace WPF triggers with the appropriate visual states, bindings, or WinUI behaviors. Put state groups on a `Page`/`UserControl` content root or a control-template root.
- Use direct Boolean `x:Bind`, existing control states, or suitable Toolkit converters before copying a WPF converter. Preserve the intended conversion and its inverse.
- Audit binding source, effective mode, and update timing. WPF editable properties that bind two-way by default need explicit `Mode=TwoWay` in WinUI.
- Put `DataContext` and element resources on the hosted view/root element. A WinUI `Window` does not expose the WPF equivalents.
- Preserve valid `PreviewKeyDown`/`PreviewKeyUp` handlers; redesign WPF mouse preview events according to their required behavior.
- Keep resource lookup out of early static initialization paths used by models, CLI entry points, or tests.

### Threading and imaging

- Map `Dispatcher.Invoke` by its completion and exception semantics. `DispatcherQueue.TryEnqueue` only reports whether work was accepted.
- Track the window/queue that owns UI state. Use that window's loaded `XamlRoot` for dialogs and its HWND or `WindowId` for the selected picker API.
- Keep image processing separate from XAML image sources. Validate pixel format, alpha, dimensions, orientation, codec options, and stream ownership.
- Preserve all required frames and metadata when replacing WPF encoders; a successful encode alone does not establish equivalent output.

## Common Migration Failures

| Symptom | What to check |
|---|---|
| A WinUI project still needs WPF assemblies | Find remaining callers, resource items, generated files, and shared-library dependencies before removing references. |
| Startup reports a Windows App SDK runtime/COM activation error | Check the chosen package identity, runtime deployment, architecture, and initialization path; do not change deployment properties blindly. |
| A window's bindings or resources do not resolve | Move view state and resources to the hosted view/root and check the binding source. |
| Edited text does not reach the view model | Check explicit `Mode=TwoWay`, update timing, and notifications. |
| A dialog or picker opens without the correct owner | Supply the owning loaded `XamlRoot`, HWND, or `WindowId` required by that API. |
| Localized strings fail during startup or tests | Check PRI generation/name, resource-map lookup, initialization timing, and the test resource accessor. |
| An installer reports a missing `.resources.dll` | Check whether the resource migration stopped producing that file, then update component and language-directory entries from actual output. |
| Unexpected `.exe`, `.deps.json`, or `.runtimeconfig.json` files appear | Check executable project references and copied output before harvesting or signing artifacts. |
| Scripts work from a solution but fail from a project | Review `$(SolutionDir)` dependencies and use a stable path base appropriate to the declaring file. |
| Packaging or signing cannot find build artifacts | Check target ordering, selected build/publish output, and stale file inventories. |
| Image quality, dimensions, or tests change | Check codec options and types, pixel/alpha representation, integer widths, and frame/metadata handling. |

## Detailed References

Read the reference that matches the current phase:

- [Project and Application Migration](references/project-and-application-migration.md) — project items, MVVM, localization, installer/build changes, tests, and compatibility.
- [Namespace and API Mapping](references/namespace-api-mapping.md) — namespaces, types, controls, properties, packages, and interop.
- [XAML Migration](references/xaml-migration.md) — markup, binding, converters, templates, styles, and resource syntax.
- [Threading and Windowing](references/threading-and-windowing.md) — queue semantics, window sizing/lifecycle, dialogs, pickers, and custom entry points.
- [Imaging Migration](references/imaging-migration.md) — streams, codecs, options, multi-frame operations, metadata, and image tests.

## Completion Checklist

- [ ] The WinUI application's project items and dependencies reflect the completed conversion.
- [ ] Startup, shutdown, window ownership, dialogs, and pickers preserve the required behavior.
- [ ] Binding updates, commands, themes, localization, keyboard access, and automation remain correct.
- [ ] Settings fixtures, CLI behavior, and both directions of any existing IPC integration remain compatible.
- [ ] Installer inventories and signing inputs match the actual build/publish output.
- [ ] Tests exercise the same image, resource, and application behavior as before the migration.
- [ ] The intended published output is validated; use `winui-dev-workflow` for Native AOT publish/run steps and `winui-packaging`'s `references/sourcegen-patterns.md` for trimming/binding/serialization guidance.
- [ ] Required repository checks pass, or any skipped check is identified with its governing restriction.
