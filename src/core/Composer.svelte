<script lang="ts" context="module">
  export type InitialEditorStateType =
    | null
    | string
    | EditorState
    | ((editor: LexicalEditor) => void);

  export type InitialConfigType = Readonly<{
    editor__DEPRECATED?: LexicalEditor | null;
    namespace: string;
    nodes?: ReadonlyArray<
      | Klass<LexicalNode>
      | {
          replace: Klass<LexicalNode>;
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          with: <T extends {new (...args: any): any}>(
            node: InstanceType<T>,
          ) => LexicalNode;
        }
    >;
    onError: (error: Error, editor: LexicalEditor) => void;
    editable?: boolean;
    theme?: EditorThemeClasses;
    editorState?: InitialEditorStateType;
  }>;
</script>

<script lang="ts">
  import {createEmptyHistoryState} from '@lexical/history';
  import {
    $getRoot as getRoot,
    $isElementNode as isElementNode,
    createEditor,
    type EditorState,
    type EditorThemeClasses,
    type Klass,
    type LexicalEditor,
    type LexicalNode,
  } from 'lexical';
  import {onMount} from 'svelte';
  import {initializeEditor} from './initializeEditor';
  import {
    createSharedEditorContext,
    setEditor,
    setHistoryStateContext,
  } from './composerContext';

  export let initialConfig: InitialConfigType;

  const {
    theme,
    namespace,
    nodes,
    onError,
    editorState: initialEditorState,
  } = initialConfig;

  const editor = createEditor({
    editable: false,
    namespace,
    nodes,
    onError: (error) => onError(error, editor),
    theme,
  });
  initializeEditor(editor, initialEditorState);
  setEditor(editor);

  setHistoryStateContext(createEmptyHistoryState());

  // allows sharing context between plugins and decorator nodes
  createSharedEditorContext();

  onMount(() => {
    const isEditable = initialConfig.editable;
    editor.setEditable(isEditable !== undefined ? isEditable : true);
  });

  export function getEditor() {
    return editor;
  }

  export function getHTML() {
    const editorState = editor.getEditorState();
    const htmlString = editorState.read(() => {
      if (typeof document === 'undefined' || typeof window === 'undefined') {
        throw new Error(
          'To use $generateHtmlFromNodes in headless mode please initialize a headless browser implementation such as JSDom before calling this function.',
        );
      }

      const container = document.createElement('div');
      const root = getRoot();
      const topLevelChildren = root.getChildren();
      for (let i = 0; i < topLevelChildren.length; i++) {
        const topLevelNode = topLevelChildren[i];
        appendNodesToHTML(editor, topLevelNode, container);
      }
      return container.innerHTML;
    });
    return htmlString;
  }

  /**
   *
   * @param {any} editor
   * @param {any} currentNode
   * @param {any} parentElement
   */
  function appendNodesToHTML(editor, currentNode, parentElement) {
    let shouldInclude = true;
    const shouldExclude =
      isElementNode(currentNode) && currentNode.excludeFromCopy('html');
    let target = currentNode;

    const children = isElementNode(target) ? target.getChildren() : [];
    const {element, after} = target.exportDOM(editor);

    if (!element) {
      return false;
    }

    const fragment = document.createDocumentFragment();

    for (let i = 0; i < children.length; i++) {
      const childNode = children[i];
      const shouldIncludeChild = appendNodesToHTML(editor, childNode, fragment);

      if (
        !shouldInclude &&
        isElementNode(currentNode) &&
        shouldIncludeChild &&
        currentNode.extractWithChild(childNode, null, 'html')
      ) {
        shouldInclude = true;
      }
    }

    if (shouldInclude && !shouldExclude) {
      element.append(fragment);
      parentElement.append(element);

      if (after) {
        const newElement = after.call(target, element);
        if (newElement) element.replaceWith(newElement);
      }
    } else {
      parentElement.append(fragment);
    }

    return shouldInclude;
  }
</script>

<slot />
