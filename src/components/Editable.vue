<template>
	<div
		id="editable"
		ref="editable"
		@keyup="remapContent"
		v-html="content"
		contenteditable="true"
		spellcheck="false"
		placeholder="Start typing..."
	></div>
</template>

<script setup lang="ts">
import { ref, nextTick } from 'vue'

const content = ref('')
const editable = ref<HTMLElement | null>(null)

interface ParentNodeWithId extends ParentNode {
	id: string;
}

function remapContent(event: Event) {
	// getting the target element
	const target = event.target as HTMLElement
	
	// checking for markdown syntax
	const { newContent, mdTagInserted } = checkForMdSyntax(target.innerHTML)

	// getting the cursor position
	const { startCharacterIndex, cursorStartNodePath } =  getCursorPosition()
	
	// setting the new content
  content.value = parseContent(newContent)

	// setting the cursor position
	nextTick(() => {
		setCursor(startCharacterIndex, cursorStartNodePath, mdTagInserted)
	})
}

function checkForMdSyntax(content: string) {
	// checking for bold syntax
	const strongRegex = /\*{2}.*?\*{2}/g
	const strongMatches = content.match(strongRegex)

	let newContent = content

	// replacing bold syntax matches with html tags
	if (strongMatches) {
		strongMatches.forEach(match => {
			const newMatch = match.replace(/\*/g, '')
			newContent = newContent.replace(match, `<strong>${newMatch}</strong>`)
		})
	}

	return { newContent, mdTagInserted: strongMatches ? true : false }
}

function parseContent(newContent: string): string {
  const parser = new DOMParser()
  const parsedDocument = parser.parseFromString(newContent, 'text/html')

  return parsedDocument.body.innerHTML
}

function getCursorPosition() {
	const selection = window.getSelection()
	const range = selection?.getRangeAt(0)

	const startCharacterIndex = range?.startOffset ? range?.startOffset : 0
	const endCharacterIndex = range?.endOffset ? range?.endOffset : 0

	const cursorStartNodePath = []
	
	let startingNode = range?.startContainer as ParentNodeWithId | null | undefined
	let startingNodeParent = range?.startContainer?.parentNode as ParentNodeWithId | null | undefined

	while(true) {
		if (startingNodeParent?.id === 'app') break
		cursorStartNodePath.unshift(Array.from(startingNodeParent?.childNodes as NodeListOf<Node>).indexOf(startingNode as Node))

		if (startingNodeParent?.id === 'editable') break
		startingNode = startingNodeParent
		startingNodeParent = startingNodeParent?.parentNode as ParentNodeWithId | null | undefined
	}

	return { startCharacterIndex, endCharacterIndex, cursorStartNodePath }
}

function setCursor( startCharacterIndex: number, cursorStartNodePath: number[], mdTagInserted: boolean) {
	const editor = editable.value

	if (!editor) return

	const range = document.createRange()
	const selection = window.getSelection()

	let startNode: Node | null | undefined = null

	cursorStartNodePath.forEach((nodeIndex, index) => {
		if (index === 0) {
			startNode = editor.childNodes[nodeIndex]
			return
		}

		startNode = startNode?.childNodes[nodeIndex]
	})

	if (mdTagInserted) {
		const startNodeParent = startNode?.parentNode as HTMLElement
		startNodeParent.innerHTML = `${startNodeParent.innerHTML}&nbsp;`
		startNode = startNodeParent.childNodes[startNodeParent.childNodes.length - 1]
		startCharacterIndex = 1
	}

	range.setStart(startNode, startCharacterIndex)

	selection?.removeAllRanges()
	selection?.addRange(range)
}
</script>

<style>
#editable {
	font-family: 'Roboto', sans-serif;
	color: #f8fafc;

	min-height: 100vh;
	width: 50vw;

	box-sizing: border-box;
	padding: 3rem 1rem 0 1rem;
}

#editable:focus {
	outline: none;
}
	

[contenteditable=true]:empty:before{
  content: attr(placeholder);
	color: #3f3f46;
	font-weight: bold;
  pointer-events: none;
  display: block; /* For Firefox */
}
</style>
