# beginWork

从之前的代码中我们可以知道,workLoopSync 判断workprogress 不断循环执行performUnitOfWork,条件是workInProgress !== null

```javascript
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

performUnitOfWork 里又是先执行的beginWork,返回next，根据next判断是否执行completeUnitOfWork

我们知道这里是一个深度优先的遍历，我们先看看beginWork 做了什么

#### packages/react-reconciler/src/ReactFiberBeginWork.new.js

```javascript
function beginWork(
  current: Fiber | null,//unitOfWork.alternate current对应的是当前dom树的节点
  workInProgress: Fiber,// workInProgress 对应的是当前的节点，即unitOfWork
  renderLanes: Lanes,
): Fiber | null {
	//跳过。。。

  if (current !== null) { // 第一次的时候current是null,这里的分支对应的是更新时，或者说除第一次的逻辑
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
		//下面的if else 貌似是为了计算 didReceiveUpdate 值，这里先略过
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        // If this is the second pass of an error or suspense boundary, there
        // may not be work scheduled on `current`, so we check for this flag.
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes,
        );
      }
      if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        // This is a special case that only exists for legacy mode.
        // See https://github.com/facebook/react/pull/19216.
        didReceiveUpdate = true;
      } else {
        // An update was scheduled on this fiber, but there are no new props
        // nor legacy context. Set this to false. If an update queue or context
        // consumer produces a changed value, it will set this to true. Otherwise,
        // the component will assume the children have not changed and bail out.
        didReceiveUpdate = false;
      }
    }
  } else {
    didReceiveUpdate = false;

    if (getIsHydrating() && isForkedChild(workInProgress)) {
      // Check if this child belongs to a list of muliple children in
      // its parent.
      //
      // In a true multi-threaded implementation, we would render children on
      // parallel threads. This would represent the beginning of a new render
      // thread for this subtree.
      //
      // We only use this for id generation during hydration, which is why the
      // logic is located in this special branch.
      const slotIndex = workInProgress.index;
      const numberOfForks = getForksAtLevel(workInProgress);
      pushTreeId(workInProgress, numberOfForks, slotIndex);
    }
  }

  // Before entering the begin phase, clear pending update priority.
  // TODO: This assumes that we're about to evaluate the component and process
  // the update queue. However, there's an exception: SimpleMemoComponent
  // sometimes bails out later in the begin phase. This indicates that we should
  // move this assignment out of the common path and into each branch.
  workInProgress.lanes = NoLanes;

  switch (workInProgress.tag) {// 首先判断workInProgress 类型，这里我们只关注HostRoot(根节点),classComponent,FunctionComponent
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes,
      );
    }
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        renderLanes,
      );
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress, renderLanes);
    case HostPortal:
      return updatePortalComponent(current, workInProgress, renderLanes);
    case ForwardRef: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === type
          ? unresolvedProps
          : resolveDefaultProps(type, unresolvedProps);
      return updateForwardRef(
        current,
        workInProgress,
        type,
        resolvedProps,
        renderLanes,
      );
    }
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
    case Mode:
      return updateMode(current, workInProgress, renderLanes);
    case Profiler:
      return updateProfiler(current, workInProgress, renderLanes);
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
    case ContextConsumer:
      return updateContextConsumer(current, workInProgress, renderLanes);
    case MemoComponent: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      // Resolve outer props first, then resolve inner props.
      let resolvedProps = resolveDefaultProps(type, unresolvedProps);
      if (__DEV__) {
        if (workInProgress.type !== workInProgress.elementType) {
          const outerPropTypes = type.propTypes;
          if (outerPropTypes) {
            checkPropTypes(
              outerPropTypes,
              resolvedProps, // Resolved for outer only
              'prop',
              getComponentNameFromType(type),
            );
          }
        }
      }
      resolvedProps = resolveDefaultProps(type.type, resolvedProps);
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        renderLanes,
      );
    }
    case SimpleMemoComponent: {
      return updateSimpleMemoComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        renderLanes,
      );
    }
    case IncompleteClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return mountIncompleteClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case SuspenseListComponent: {
      return updateSuspenseListComponent(current, workInProgress, renderLanes);
    }
    case ScopeComponent: {
      if (enableScopeAPI) {
        return updateScopeComponent(current, workInProgress, renderLanes);
      }
      break;
    }
    case OffscreenComponent: {
      return updateOffscreenComponent(current, workInProgress, renderLanes);
    }
    case LegacyHiddenComponent: {
      return updateLegacyHiddenComponent(current, workInProgress, renderLanes);
    }
    case CacheComponent: {
      if (enableCache) {
        return updateCacheComponent(current, workInProgress, renderLanes);
      }
      break;
    }
  }

  throw new Error(
    `Unknown unit of work tag (${workInProgress.tag}). This error is likely caused by a bug in ` +
      'React. Please file an issue.',
  );
}
```

HostRoot  updateHostRoot

```javascript
function updateHostRoot(current, workInProgress, renderLanes) {
  pushHostRootContext(workInProgress);
  const updateQueue = workInProgress.updateQueue;

  if (current === null || updateQueue === null) {
    throw new Error(
      'If the root does not have an updateQueue, we should have already ' +
        'bailed out. This error is likely caused by a bug in React. Please ' +
        'file an issue.',
    );
  }

  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState.element;
  cloneUpdateQueue(current, workInProgress);// 把current的updateQueue 复制到workInProgress
  processUpdateQueue(workInProgress, nextProps, null, renderLanes);// 解析updateQueue,计算新的state,这两部在classComponent中也有，稍后再说
  const nextState = workInProgress.memoizedState;

  const root: FiberRoot = workInProgress.stateNode;
	// 跳过。。。

  // Caution: React DevTools currently depends on this property
  // being called "element".
  const nextChildren = nextState.element;
  if (nextChildren === prevChildren) {
    resetHydrationState();
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  if (root.isDehydrated && enterHydrationState(workInProgress)) {
	// SSR相关，跳过。。。
  } else {
    // Otherwise reset hydration state in case we aborted and resumed another
    // root.
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);// 调用reconcileChildren，开始处理子节点，同样classComponet中也会有这些逻辑，稍后再说
    resetHydrationState();
  }
  return workInProgress.child; // 注意这里返回了child,即 performUnitWork 中的next,如果不为null,会一直深度递归直到最深的那个子节点，注意是子节点
}
```

看看updateClassComponent

```javascript
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes,
) {
  //跳过。。。

  // Push context providers early to prevent context stack mismatches.
  // During mounting we don't know the child context yet as the instance doesn't exist.
  // We will invalidate the child context in finishClassComponent() right after rendering.
  let hasContext;
  if (isLegacyContextProvider(Component)) {// context 老的写法
    hasContext = true;
    pushLegacyContextProvider(workInProgress);
  } else {
    hasContext = false;
  }
  prepareToReadContext(workInProgress, renderLanes);

  const instance = workInProgress.stateNode;// 阅读后面的代码直到，构建好classComponent的实例后，会挂在workInProgress.stateNode 上
  let shouldUpdate;
  if (instance === null) {// 初次挂载
    if (current !== null) { // current 是当前的dom树上对应的节点，这里的逻辑对应错误的情况，所以重置下
      // A class component without an instance only mounts if it suspended
      // inside a non-concurrent tree, in an inconsistent state. We want to
      // treat it like a new mount, even though an empty version of it already
      // committed. Disconnect the alternate pointers.
      current.alternate = null;
      workInProgress.alternate = null;
      // Since this is conceptually a new fiber, schedule a Placement effect
      workInProgress.flags |= Placement;
    }
    // In the initial pass we might need to construct the instance.
    constructClassInstance(workInProgress, Component, nextProps); // construct 即 new 这个component, 创建出组件的实例，执行 constructor,初始化
    mountClassInstance(workInProgress, Component, nextProps, renderLanes);// 执行classComponent 的componentDidmount
    shouldUpdate = true;//因为是首次的挂载，从无到有。shouldUpdate 肯定要设置成true
  } else if (current === null) {
    // In a resume, we'll already have an instance we can reuse.
    shouldUpdate = resumeMountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderLanes,
    );
  } else {
    shouldUpdate = updateClassInstance( // 对应的是更新逻辑，执行updateClassInstance 内部是cloneUpdateQueue，processUpdateQueue，计算出新的state,然后调用instance.shouldComponentUpdate ，计算shouldUpdate
      计算出来的新的state,props,会存在workInProgress.memoizedState 
      current,
      workInProgress,
      Component,
      nextProps,
      renderLanes,
    );
  }
  const nextUnitOfWork = finishClassComponent(// 
    current,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderLanes,
  );
	// 跳过。。。
  return nextUnitOfWork;
}
```

```javascript
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderLanes: Lanes,
) {
  // Refs should update even if shouldComponentUpdate returns false
  markRef(current, workInProgress);

  const didCaptureError = (workInProgress.flags & DidCapture) !== NoFlags;

  if (!shouldUpdate && !didCaptureError) {
    // Context providers should defer to sCU for rendering
    if (hasContext) {
      invalidateContextProvider(workInProgress, Component, false);
    }
		//出错了，跳过已经完成的work。。。
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  const instance = workInProgress.stateNode;

  // Rerender
  ReactCurrentOwner.current = workInProgress;
  let nextChildren;
  if (
    didCaptureError &&
    typeof Component.getDerivedStateFromError !== 'function'
  ) {
    // If we captured an error, but getDerivedStateFromError is not defined,
    // unmount all the children. componentDidCatch will schedule an update to
    // re-render a fallback. This is temporary until we migrate everyone to
    // the new API.
    // TODO: Warn in a future release.
    nextChildren = null;

    if (enableProfilerTimer) {
      stopProfilerTimerIfRunning(workInProgress);
    }
  } else {
    if (enableSchedulingProfiler) {
      markComponentRenderStarted(workInProgress);
    }
    if (__DEV__) {
      setIsRendering(true);
      nextChildren = instance.render();
      if (
        debugRenderPhaseSideEffectsForStrictMode &&
        workInProgress.mode & StrictLegacyMode
      ) {
        setIsStrictModeForDevtools(true);
        try {
          instance.render(); 
        } finally {
          setIsStrictModeForDevtools(false);
        }
      }
      setIsRendering(false);
    } else {
      nextChildren = instance.render(); // 执行 render 返回值赋给 nextChildren
    }
    if (enableSchedulingProfiler) {
      markComponentRenderStopped();
    }
  }

  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  if (current !== null && didCaptureError) {
    // If we're recovering from an error, reconcile without reusing any of
    // the existing children. Conceptually, the normal children and the children
    // that are shown on error are two different sets, so we shouldn't reuse
    // normal children even if their identities match.
    forceUnmountCurrentAndReconcile(
      current,
      workInProgress,
      nextChildren,
      renderLanes,
    );
  } else {
    reconcileChildren(current, workInProgress, nextChildren, renderLanes); // reconcileChildren ,构建|更新子节点
  }

  // Memoize state using the values we just used to render.
  // TODO: Restructure so we never read values from the instance.
  workInProgress.memoizedState = instance.state;// 保存state,保存workInProgress.memoizedState

  // The context might have changed so we need to recalculate it.
  if (hasContext) {
    invalidateContextProvider(workInProgress, Component, true);
  }

  return workInProgress.child; // 返回child
}
```

针对classComponent ，如果是完全的初次挂载，会执行construct 构造实例，然后mount,执行componentDidMount ,然后执行updateClassInstance。 其中逻辑是把current上对应的节点updateQueue复制过来，cloneUpdateQueue,processUpdateQueue

计算newState,计算 shouldUpdate，然后执行finishClassComponent ,执行render 获取children 然后再执行 reconcileChildren 构造子节点

### cloneUpdateQueue  processUpdateQueue









