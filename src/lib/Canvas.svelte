<script>
	import { onMount, onDestroy, createEventDispatcher } from 'svelte';
	import paper from 'paper';
	import Frame from './Frame.svelte';

	export let file;
	export let id;

	export let width = '500px';
	export let height = '500px';

	export let actions = [];
	/* =====================================================
        SHARED STATE / GLOBALS
    ======================================================= */
	let canvas;
	let project;

	// MULTI-SELECTION: track multiple groups
	let selectedGroups = [];
	let selectedGroup = null; // if exactly 1 group
	let selectedGroupType = null; // 'shape' | 'text'
	let isEditingShape = false;
	let guideBox = null;

	let selectedSegments = []; // Reactive array for selected segments
	let dragStartPositions = {};

	let isPenToolActive = false;
	let penVertices = [];
	let penCursor = null;

	/* =====================================================
        MENU CONFIGURATION
    ======================================================= */
	$: topMenuActions = [
		{
			name: 'Save',
			icon: 'mdi:content-save-outline',
			show: () => selectedGroups.length === 0,
			action: () => doSave()
		},
		{
			name: 'Cancel',
			icon: 'mdi:close-circle-outline',
			show: () => !isPenToolActive && selectedGroupType,
			action: () => doCancel()
		},
		{
			name: 'Edit Text',
			icon: 'mdi:pencil-outline',
			show: () => selectedGroupType === 'text' && !isPenToolActive,
			action: () => textOps.editText()
		},
		{
			name: 'Edit Vector Points',
			icon: 'mdi:pencil-outline',
			show: () => selectedGroupType === 'shape' && !isEditingShape && !isPenToolActive,
			action: () => enterShapeEditMode()
		},
		{
			name: 'Finish Vector Editing',
			icon: 'mdi:check-circle-outline',
			show: () => selectedGroupType === 'shape' && isEditingShape && !isPenToolActive,
			action: () => exitShapeEditMode()
		},
		{
			name: 'Finish Path',
			icon: 'mdi:check-circle-outline',
			show: () => isPenToolActive,
			action: () => vectorOps.finalizePath()
		},
		{
			name: 'Clone',
			icon: 'mdi:content-copy',
			show: () => !isPenToolActive && selectedGroups.length > 0,
			action: () => cloneGroups()
		},
		{
			name: 'Flip Horizontal',
			icon: 'mdi:flip-horizontal',
			show: () => !isPenToolActive && selectedGroups.length > 0,
			action: () => flipOps.flipHorizontal()
		},
		{
			name: 'Flip Vertical',
			icon: 'mdi:flip-vertical',
			show: () => !isPenToolActive && selectedGroups.length > 0,
			action: () => flipOps.flipVertical()
		},
		{
			name: 'Angled Corner',
			icon: 'mdi:vector-polyline',
			show: () => isEditingShape && selectedSegments.length > 0,
			action: () => makeAngledCorners()
		},
		{
			name: 'Asymmetric Corner',
			icon: 'mdi:vector-bezier',
			show: () => isEditingShape && selectedSegments.length > 0,
			action: () => makeAsymmetricCorners()
		},
		{
			name: 'Symmetric Corner',
			icon: 'mdi:vector-curve',
			show: () => isEditingShape && selectedSegments.length > 0,
			action: () => makeSymmetricCorners()
		},
		{
			name: 'Opposing Asymmetric Corner',
			icon: 'mdi:vector-line',
			show: () => isEditingShape && selectedSegments.length > 0,
			action: () => makeOpposingAsymmetricCorners()
		},
		{
			name: 'Delete',
			icon: 'mdi:delete-outline',
			show: () => selectedGroups.length > 0 && !isPenToolActive,
			action: () => selectionOps.deleteItem()
		},
		{
			name: () => selectedGroupType || 'None',
			icon: '',
			show: () => selectedGroups.length === 1 && !isPenToolActive,
			action: () => selectionOps.deselectAll()
		}
	];

	let shapeSubmenuItems = [
		{ name: 'Circle', icon: 'mdi:circle-outline', action: () => shapeOps.addCircle() },
		{ name: 'Square', icon: 'mdi:square-outline', action: () => shapeOps.addSquare() },
		{ name: 'Triangle', icon: 'mdi:triangle-outline', action: () => shapeOps.addTriangle() },
		{ name: 'Line', icon: 'mdi:minus', action: () => shapeOps.addLine() },
		{ name: 'Polygon', icon: 'mdi:hexagon-outline', action: () => shapeOps.addPolygon() },
		{ name: 'Star', icon: 'mdi:star-outline', action: () => shapeOps.addStar() }
	];

	let possibleActions = [
		{
			type: 'upload',
			name: 'Add File',
			icon: 'mdi:file-image-plus-outline',
			action: () => fileOps.addFile()
		},
		{
			type: 'download',
			name: 'Download File',
			icon: 'mdi:download-outline',
			action: () => fileOps.downloadFile()
		},
		{ type: 'text', name: 'Add Text', icon: 'mdi:format-text', action: () => textOps.addText() },
		{
			type: 'shape',
			name: 'Add Shape',
			icon: 'mdi:shape-outline',
			hasSubmenu: true,
			submenuItems: shapeSubmenuItems
		},
		{
			type: 'vector',
			name: 'Add Vector',
			icon: 'mdi:fountain-pen-tip',
			action: () => vectorOps.activatePenTool()
		}
	];

	let verticalMenuItems = [
		...possibleActions.filter((action) =>
			actions.some((selection) => selection.type === action.type)
		),
		...actions.filter((action) => action.type === 'custom')
	];

	const eventDispatcher = createEventDispatcher();

	$: console.log('Canvas Actions:', verticalMenuItems);
	/* =====================================================
        HELPERS
    ======================================================= */
	function createCircle(
		center,
		{ radius = 5, fillColor = 'white', strokeColor = 'black', strokeWidth = 1, data = {} } = {}
	) {
		const circle = new paper.Path.Circle({
			center,
			radius,
			fillColor,
			strokeColor,
			strokeWidth
		});
		circle.data = { ...data };
		return circle;
	}

	function createLine(
		from,
		to,
		{ strokeColor = 'black', strokeWidth = 2, dashArray = [], data = {} } = {}
	) {
		const line = new paper.Path.Line({
			from,
			to,
			strokeColor,
			strokeWidth,
			dashArray
		});
		line.data = { ...data };
		return line;
	}

	function finalizeShapeCreation(shapePath, type) {
		shapePath.data = { type };
		shapePath.segments.forEach((segment) => {
			if (!segment.data) {
				segment.data = {};
			}
		});
		const group = new paper.Group([shapePath]);
		group.data = { isShapeGroup: true, editMode: false };

		selectionOps.deselectAll();
		selectionOps.selectGroup(group, type);
		project.view.draw();
	}
	/* =====================================================
        SHAPE OPS
    ======================================================= */
	const shapeOps = {
		addCircle() {
			const shape = new paper.Path.Circle({
				center: project.view.center,
				radius: 50,
				fillColor: '#3498db',
				strokeColor: '#2980b9',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'circle');
		},
		addSquare() {
			const size = 100;
			const shape = new paper.Path.Rectangle({
				point: project.view.center.subtract([size / 2, size / 2]),
				size: [size, size],
				fillColor: '#e67e22',
				strokeColor: '#d35400',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'rectangle');
		},
		addTriangle() {
			const shape = new paper.Path.RegularPolygon({
				center: project.view.center,
				sides: 3,
				radius: 60,
				fillColor: '#9b59b6',
				strokeColor: '#8e44ad',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'triangle');
		},
		addLine() {
			const shape = new paper.Path.Line({
				from: project.view.center.subtract([50, 0]),
				to: project.view.center.add([50, 0]),
				strokeColor: '#2c3e50',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'line');
		},
		addPolygon() {
			const shape = new paper.Path.RegularPolygon({
				center: project.view.center,
				sides: 6,
				radius: 50,
				fillColor: '#f1c40f',
				strokeColor: '#f39c12',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'polygon');
		},
		addStar() {
			const shape = new paper.Path.Star({
				center: project.view.center,
				points: 5,
				radius1: 30,
				radius2: 60,
				fillColor: '#e74c3c',
				strokeColor: '#c0392b',
				strokeWidth: 2
			});
			finalizeShapeCreation(shape, 'star');
		}
	};

	/* =====================================================
        FILE OPS
    ======================================================= */
	const fileOps = {
		addFile(svgString, size) {
			if (svgString) {
				try {
					paper.project.importSVG(svgString, {
						onLoad: (item) => {
							const group = new paper.Group([item]);
							group.data = { isShapeGroup: true, editMode: false };
							group.scale(size[0] / group.bounds.width, size[1] / group.bounds.height);
							paper.project.activeLayer.addChild(group);
							initializeImportedSVGData(group);
							paper.view.draw();
						},
						onError: (message) => {
							console.error('Error importing SVG:', message);
						}
					});
				} catch (error) {
					console.error('Failed to import SVG:', error);
				}
			}
			const input = document.createElement('input');
			input.type = 'file';
			input.accept = 'image/svg+xml';
			input.onchange = (event) => {
				const f = event.target.files[0];
				if (f) {
					const reader = new FileReader();
					reader.onload = (e) => {
						const svgContent = e.target.result;
						try {
							paper.project.importSVG(svgContent, {
								onLoad: (item) => {
									const group = new paper.Group([item]);
									group.data = { isShapeGroup: true, editMode: false };
									paper.project.activeLayer.addChild(group);
									initializeImportedSVGData(group);
									paper.view.draw();
								},
								onError: (message) => {
									console.error('Error importing SVG:', message);
								}
							});
						} catch (error) {
							console.error('Failed to import SVG:', error);
						}
					};
					reader.readAsText(f);
				}
			};
			input.click();
		},
		downloadFile() {
			if (!project) return;
			const svg = project.exportSVG({ asString: true });
			const blob = new Blob([svg], { type: 'image/svg+xml;charset=utf-8' });
			const url = URL.createObjectURL(blob);
			const a = document.createElement('a');
			a.href = url;
			a.download = 'canvas.svg';
			document.body.appendChild(a);
			a.click();
			document.body.removeChild(a);
			URL.revokeObjectURL(url);
		}
	};

	function initializeImportedSVGData(group) {
		if (group instanceof paper.Group) {
			group.children.forEach((child) => {
				if (child instanceof paper.Path) {
					child.segments.forEach((segment) => {
						if (!segment.data) {
							segment.data = {};
						}
						// Initialize default handle lengths
						if (segment.handleIn.length === 0 && segment.handleOut.length === 0) {
							segment.handleIn = new paper.Point(-50, 0);
							segment.handleOut = new paper.Point(50, 0);
						}
					});
				} else if (child instanceof paper.PointText) {
					child.data = { isText: true };
				}
			});
		}
	}

	/* =====================================================
        TEXT OPS
    ======================================================= */
	const textOps = {
		addText() {
			const textContent = prompt('Enter text:', 'Hello, World!');
			if (textContent !== null) {
				const textItem = new paper.PointText({
					point: project.view.center,
					content: textContent,
					fillColor: 'black',
					fontSize: 24
				});
				textItem.data = { isText: true };
				const group = new paper.Group([textItem]);
				group.data = { isShapeGroup: true, editMode: false };

				selectionOps.deselectAll();
				selectionOps.selectGroup(group, 'text');
				project.view.draw();
			}
		},
		editText() {
			if (selectedGroup && selectedGroupType === 'text') {
				const textItem = selectedGroup.children[0];
				const newText = prompt('Edit text:', textItem.content);
				if (newText !== null) {
					textItem.content = newText;
				}
			}
		}
	};

	/* =====================================================
        VECTOR OPS
    ======================================================= */
	let mainTool, penTool;
	let tempPath = null;

	const vectorOps = {
		activatePenTool() {
			if (isEditingShape) {
				exitShapeEditMode();
			}
			selectionOps.deselectAll();

			isPenToolActive = true;
			penTool.activate();

			if (!penCursor) {
				penCursor = createCircle(project.view.center, {
					radius: 4,
					fillColor: 'rgba(0,0,0,0.5)',
					strokeColor: 'white',
					strokeWidth: 1,
					data: { nonInteractive: true }
				});
			}
			penCursor.visible = true;
		},

		finalizePath() {
			if (tempPath && tempPath.segments.length > 1) {
				tempPath.segments.forEach((segment) => {
					if (!segment.data) {
						segment.data = {};
					}
				});
				finalizeShapeCreation(tempPath, 'customVector');
			} else {
				tempPath?.remove();
			}
			tempPath = null;

			penVertices.forEach((c) => c.remove());
			penVertices = [];

			if (penCursor) {
				penCursor.visible = false;
			}
			isPenToolActive = false;
			mainTool.activate();
		}
	};

	/* =====================================================
        FLIP OPS
    ======================================================= */
	const flipOps = {
		flipHorizontal() {
			if (selectedGroups.length === 0) return;
			selectedGroups.forEach((group) => {
				group.scale(-1, 1, group.position);
			});
			boundingBoxOps.updateGuideBoxForSelection(selectedGroups);
			project.view.draw();
		},
		flipVertical() {
			if (selectedGroups.length === 0) return;
			selectedGroups.forEach((group) => {
				group.scale(1, -1, group.position);
			});
			boundingBoxOps.updateGuideBoxForSelection(selectedGroups);
			project.view.draw();
		}
	};

	/* =====================================================
        GENERATION OPS
    ======================================================= */
	const generationOps = {
		async generateImage() {
			const promptText = prompt('Enter a prompt for image generation:');
			if (promptText !== null) {
				// eventDispatcher('generateImage', { prompt: promptText });

				try {
					const response = await fetch(`/api/ai/svg`, {
						method: 'post',
						headers: {
							'Content-Type': 'application/json'
						},
						body: JSON.stringify({ prompt: promptText, projectId: id })
					});
					const result = await response.json();
					fileOps.addFile(result.svg, [500, 500]);
					console.log('Success:', result);
					return result;
				} catch (error) {
					console.error('Error:', error);
				}
			}
		}
	};

	/* =====================================================
        SELECTION OPS
    ======================================================= */
	const selectionOps = {
		selectGroup(group, type, shiftHeld = false) {
			if (shiftHeld) {
				const idx = selectedGroups.indexOf(group);
				if (idx >= 0) {
					selectedGroups.splice(idx, 1);
				} else {
					selectedGroups.push(group);
				}
				// Reassign to trigger reactivity
				selectedGroups = [...selectedGroups];
			} else {
				if (!selectedGroups.includes(group)) {
					this.deselectAll();
					selectedGroups = [group];
					// Reassign to trigger reactivity
					selectedGroups = [...selectedGroups];
				}
			}

			// Update selectedGroup and type
			if (selectedGroups.length === 1) {
				selectedGroup = selectedGroups[0];
				selectedGroupType = type;
			} else {
				selectedGroup = null;
				selectedGroupType = null;
			}

			boundingBoxOps.createGuideBoxForSelection(selectedGroups);
		},

		deselectAll() {
			boundingBoxOps.removeGuideBox();
			selectedGroups = [];
			selectedGroup = null;
			selectedGroupType = null;
			// Reassign to trigger reactivity
			selectedGroups = [];
			project.view.draw();
		},

		deleteItem() {
			if (selectedGroups.length === 0) return;
			selectedGroups.forEach((group) => group.remove());
			this.deselectAll();
		}
	};

	/* =====================================================
        EDITING OPS (Shape Anchor Points)
    ======================================================= */
	function enterShapeEditMode() {
		if (!selectedGroup || selectedGroupType !== 'shape') return;
		const shapePath = selectedGroup.children[0];
		if (!(shapePath instanceof paper.Path)) return;

		isEditingShape = true;
		boundingBoxOps.removeGuideBox();
		selectedGroup.data.editMode = true;
		createAnchorCircles(shapePath);
	}

	/* =====================================================
    EDITING OPS (Shape Anchor Points)
======================================================= */
	function exitShapeEditMode() {
		isEditingShape = false;
		if (selectedGroup) {
			selectedGroup.data.editMode = false;
		}
		removeAllSegmentHandles(); // Now removes anchor circles as well
		selectedSegments = [];
		if (selectedGroup) {
			boundingBoxOps.createGuideBoxForSelection([selectedGroup]);
		}
	}

	function removeAllSegmentHandles() {
		const shapePath = selectedGroup.children[0];
		if (!(shapePath instanceof paper.Path)) return;

		// Remove handleIn and handleOut circles and lines for selected segments
		selectedSegments.forEach((segIndex) => {
			removeHandlesForSegment(shapePath, segIndex);
		});

		// Additionally, remove all anchor circles from all segments
		shapePath.segments.forEach((segment, index) => {
			if (segment.data?.anchor) {
				segment.data.anchor.remove(); // Remove the anchor circle from the canvas
				segment.data.anchor = null; // Clear the reference
			}
		});

		selectedSegments = [];
	}

	function createAnchorCircles(shapePath) {
		removeAllSegmentHandles();
		selectedSegments = []; // Reset selection

		shapePath.segments.forEach((segment, index) => {
			if (!segment.data) segment.data = {};
			const anchorCircle = createCircle(segment.point, {
				...getAnchorStyle(false),
				data: { segmentIndex: index, handleType: 'anchor' }
			});
			segment.data.anchor = anchorCircle; // Store anchor in segment data

			let wasDragged = false;

			anchorCircle.onMouseDown = (ev) => {
				ev.stopPropagation();
				wasDragged = false;
				if (selectedSegments.includes(index)) {
					storeDragStartPositions(shapePath);
				} else {
					dragStartPositions = {
						[index]: shapePath.segments[index].point.clone()
					};
				}
			};

			anchorCircle.onMouseDrag = (ev) => {
				wasDragged = true;
				const seg = shapePath.segments[index];
				const oldPos = dragStartPositions[index];
				if (!oldPos) return;
				const delta = ev.point.subtract(oldPos);

				let segsToMove = [...selectedSegments];
				if (!selectedSegments.includes(index)) {
					segsToMove = [index];
				}
				segsToMove.forEach((sidx) => {
					const startPos = dragStartPositions[sidx];
					shapePath.segments[sidx].point = startPos.add(delta);
					updateSegmentHandles(shapePath, sidx);
				});
				project.view.draw();
			};

			anchorCircle.onMouseUp = (ev) => {
				if (!wasDragged) {
					const shiftHeld = ev.event.shiftKey;
					handleAnchorClick(shapePath, index, shiftHeld);
				}
			};

			// Add anchorCircle as a child of the group to ensure it moves with the shape
			shapePath.parent.addChild(anchorCircle);
		});
	}

	function getAnchorStyle(isSelected) {
		return isSelected
			? { radius: 5, fillColor: 'orange', strokeColor: 'black', strokeWidth: 1 }
			: { radius: 5, fillColor: 'yellow', strokeColor: 'black', strokeWidth: 1 };
	}

	function storeDragStartPositions(shapePath) {
		dragStartPositions = {};
		selectedSegments.forEach((idx) => {
			dragStartPositions[idx] = shapePath.segments[idx].point.clone();
		});
	}

	function handleAnchorClick(shapePath, segIndex, shiftHeld) {
		if (!shiftHeld) {
			clearAllAnchorSelections();
		}

		const isSelected = selectedSegments.includes(segIndex);

		if (isSelected) {
			// Deselect the segment
			selectedSegments = selectedSegments.filter((idx) => idx !== segIndex);
			removeHandlesForSegment(shapePath, segIndex);
			setAnchorSelected(segIndex, false);
		} else {
			// Select the segment
			selectedSegments = [...selectedSegments, segIndex];
			createHandlesForSegment(shapePath, segIndex);
			setAnchorSelected(segIndex, true);
		}

		// // After updating selection, redraw the view
		// project.view.draw();
	}

	function clearAllAnchorSelections() {
		selectedSegments.forEach((segIndex) => {
			removeHandlesForSegment(selectedGroup.children[0], segIndex);
			setAnchorSelected(segIndex, false);
		});
		selectedSegments = [];
	}

	function createHandlesForSegment(shapePath, segIndex) {
		const segment = shapePath.segments[segIndex];
		const anchorPos = segment.point;
		if (!segment.data) segment.data = {};
		const mode = segment.data.mode;

		removeHandlesForSegment(shapePath, segIndex);

		// HandleIn
		if (segment.handleIn.length !== 0) {
			const handleInPos = anchorPos.add(segment.handleIn);
			const handleInCircle = createCircle(handleInPos, {
				radius: 4,
				fillColor:
					mode === 'symmetric' ? 'green' : mode === 'opposingAsymmetric' ? 'purple' : 'blue',
				data: { segmentIndex: segIndex, handleType: 'handleIn' }
			});
			handleInCircle.onMouseDown = (ev) => ev.stopPropagation();
			handleInCircle.onMouseDrag = (ev) => {
				segment.handleIn = ev.point.subtract(anchorPos);
				if (mode === 'symmetric') {
					segment.handleOut = segment.handleIn.clone().multiply(-1);
				} else if (mode === 'opposingAsymmetric') {
					const inDir = segment.handleIn.normalize();
					const outLen = segment.handleOut.length;
					segment.handleOut = inDir.multiply(-outLen);
				}
				updateSegmentHandles(shapePath, segIndex);
				project.view.draw();
			};

			const handleInLine = createLine(anchorPos, handleInPos, {
				strokeColor: 'blue',
				dashArray: [2, 2],
				data: { segmentIndex: segIndex, handleType: 'handleInLine' }
			});

			// Store handles in segment data
			segment.data.handleInCircle = handleInCircle;
			segment.data.handleInLine = handleInLine;

			// Add to group
			shapePath.parent.addChild(handleInLine);
			shapePath.parent.addChild(handleInCircle);
		}

		// HandleOut
		if (segment.handleOut.length !== 0) {
			const handleOutPos = anchorPos.add(segment.handleOut);
			const handleOutCircle = createCircle(handleOutPos, {
				radius: 4,
				fillColor:
					mode === 'symmetric' ? 'green' : mode === 'opposingAsymmetric' ? 'purple' : 'red',
				data: { segmentIndex: segIndex, handleType: 'handleOut' }
			});
			handleOutCircle.onMouseDown = (ev) => ev.stopPropagation();
			handleOutCircle.onMouseDrag = (ev) => {
				segment.handleOut = ev.point.subtract(anchorPos);
				if (mode === 'symmetric') {
					segment.handleIn = segment.handleOut.clone().multiply(-1);
				} else if (mode === 'opposingAsymmetric') {
					const outDir = segment.handleOut.normalize();
					const inLen = segment.handleIn.length;
					segment.handleIn = outDir.multiply(-inLen);
				}
				updateSegmentHandles(shapePath, segIndex);
				project.view.draw();
			};

			const handleOutLine = createLine(anchorPos, handleOutPos, {
				strokeColor: 'red',
				dashArray: [2, 2],
				data: { segmentIndex: segIndex, handleType: 'handleOutLine' }
			});

			// Store handles in segment data
			segment.data.handleOutCircle = handleOutCircle;
			segment.data.handleOutLine = handleOutLine;

			// Add to group
			shapePath.parent.addChild(handleOutLine);
			shapePath.parent.addChild(handleOutCircle);
		}
	}

	function removeHandlesForSegment(shapePath, segIndex) {
		const segment = shapePath.segments[segIndex];
		if (!segment || !segment.data) return;

		// Remove HandleIn
		if (segment.data.handleInCircle) {
			segment.data.handleInCircle.remove();
			segment.data.handleInLine.remove();
			segment.data.handleInCircle = null;
			segment.data.handleInLine = null;
		}

		// Remove HandleOut
		if (segment.data.handleOutCircle) {
			segment.data.handleOutCircle.remove();
			segment.data.handleOutLine.remove();
			segment.data.handleOutCircle = null;
			segment.data.handleOutLine = null;
		}
	}

	function updateSegmentHandles(shapePath, segIndex) {
		const segment = shapePath.segments[segIndex];
		const anchorPos = segment.point;

		// Get the parent group
		const group = shapePath.parent;

		// Convert the global anchor position to the group's local coordinates
		const anchorPosLocal = group.globalToLocal(anchorPos);

		// Update Anchor Position in local coordinates
		if (segment.data.anchor) {
			segment.data.anchor.position = anchorPosLocal;
		}

		// Update HandleIn
		if (segment.handleIn.length !== 0 && segment.data.handleInCircle) {
			const inPos = anchorPos.add(segment.handleIn);
			const inPosLocal = group.globalToLocal(inPos);
			segment.data.handleInCircle.position = inPosLocal;

			// **Update Both Points of handleInLine**
			segment.data.handleInLine.segments[0].point = anchorPosLocal; // Start point
			segment.data.handleInLine.segments[1].point = inPosLocal; // End point
		} else if (segment.data.handleInCircle) {
			segment.data.handleInCircle.remove();
			segment.data.handleInLine.remove();
			segment.data.handleInCircle = null;
			segment.data.handleInLine = null;
		}

		// Update HandleOut
		if (segment.handleOut.length !== 0 && segment.data.handleOutCircle) {
			const outPos = anchorPos.add(segment.handleOut);
			const outPosLocal = group.globalToLocal(outPos);
			segment.data.handleOutCircle.position = outPosLocal;

			// **Update Both Points of handleOutLine**
			segment.data.handleOutLine.segments[0].point = anchorPosLocal; // Start point
			segment.data.handleOutLine.segments[1].point = outPosLocal; // End point
		} else if (segment.data.handleOutCircle) {
			segment.data.handleOutCircle.remove();
			segment.data.handleOutLine.remove();
			segment.data.handleOutCircle = null;
			segment.data.handleOutLine = null;
		}
	}

	function setAnchorSelected(segIndex, isSelected) {
		const shapePath = selectedGroup.children[0];
		const segment = shapePath.segments[segIndex];
		if (!segment || !segment.data || !segment.data.anchor) {
			console.warn(`Anchor circle for segment ${segIndex} not found`);
			return;
		}
		const circle = segment.data.anchor;
		const style = getAnchorStyle(isSelected);
		circle.fillColor = style.fillColor;
		circle.strokeColor = style.strokeColor;
		circle.strokeWidth = style.strokeWidth;
		// Redraw to apply color changes
		project.view.draw();
	}

	function makeAngledCorners() {
		if (!selectedGroup) return;
		const shapePath = selectedGroup.children[0];
		selectedSegments.forEach((segIndex) => {
			const seg = shapePath.segments[segIndex];
			if (!seg) return;
			if (!seg.data) seg.data = {};
			seg.data.mode = 'angled';
			seg.handleIn = new paper.Point(0, 0);
			seg.handleOut = new paper.Point(0, 0);
			updateSegmentHandles(shapePath, segIndex);
			createHandlesForSegment(shapePath, segIndex);
		});
		project.view.draw();
	}

	function makeAsymmetricCorners() {
		if (!selectedGroup) return;
		const shapePath = selectedGroup.children[0];
		selectedSegments.forEach((segIndex) => {
			const seg = shapePath.segments[segIndex];
			if (!seg) return;
			if (!seg.data) seg.data = {};
			seg.data.mode = 'asymmetric';
			const outLen = seg.handleOut.length;
			const inLen = seg.handleIn.length;
			if (outLen > 0) {
				seg.handleIn = seg.handleOut.clone().multiply(-1);
			} else if (inLen > 0) {
				seg.handleOut = seg.handleIn.clone().multiply(-1);
			} else {
				seg.handleOut = new paper.Point(50, 0);
				seg.handleIn = new paper.Point(-50, 0);
			}
			updateSegmentHandles(shapePath, segIndex);
			createHandlesForSegment(shapePath, segIndex);
		});
		project.view.draw();
	}

	function makeSymmetricCorners() {
		if (!selectedGroup) return;
		const shapePath = selectedGroup.children[0];
		selectedSegments.forEach((segIndex) => {
			const seg = shapePath.segments[segIndex];
			if (!seg) return;
			if (!seg.data) seg.data = {};
			seg.data.mode = 'symmetric';
			if (seg.handleOut.length === 0) {
				seg.handleOut = new paper.Point(50, 0);
			}
			seg.handleIn = seg.handleOut.clone().multiply(-1);
			updateSegmentHandles(shapePath, segIndex);
			createHandlesForSegment(shapePath, segIndex);
		});
		project.view.draw();
	}

	function makeOpposingAsymmetricCorners() {
		if (!selectedGroup) return;
		const shapePath = selectedGroup.children[0];
		selectedSegments.forEach((segIndex) => {
			const seg = shapePath.segments[segIndex];
			if (!seg) return;
			if (!seg.data) seg.data = {};
			seg.data.mode = 'opposingAsymmetric';
			if (seg.handleOut.length === 0 && seg.handleIn.length === 0) {
				seg.handleOut = new paper.Point(50, 0);
				seg.handleIn = new paper.Point(-50, 0);
			} else {
				if (seg.handleOut.length === 0) {
					seg.handleOut = seg.handleIn.clone().multiply(-1);
				}
				if (seg.handleIn.length === 0) {
					seg.handleIn = seg.handleOut.clone().multiply(-1);
				}
			}
			updateSegmentHandles(shapePath, segIndex);
			createHandlesForSegment(shapePath, segIndex);
		});
		project.view.draw();
	}

	/* =====================================================
        BOUNDING BOX OPS
    ======================================================= */
	const boundingBoxOps = {
		createGuideBoxForSelection(groups) {
			this.removeGuideBox();
			if (!groups || groups.length === 0) return;

			let combinedBounds = null;
			groups.forEach((group, i) => {
				if (i === 0) {
					combinedBounds = group.bounds.clone();
				} else {
					combinedBounds = combinedBounds.unite(group.bounds);
				}
			});

			guideBox = new paper.Group();
			const rect = new paper.Path.Rectangle(combinedBounds);
			rect.strokeColor = 'black';
			rect.dashArray = [4, 4];
			rect.data = { nonInteractive: true };
			guideBox.addChild(rect);

			const corners = [
				combinedBounds.topLeft,
				combinedBounds.topRight,
				combinedBounds.bottomRight,
				combinedBounds.bottomLeft
			];
			corners.forEach((corner, index) => {
				const cornerHandle = createCircle(corner, {
					radius: 5,
					fillColor: 'white',
					strokeColor: 'black',
					data: { isHandle: true, corner: index }
				});
				guideBox.addChild(cornerHandle);
			});

			const topCenter = combinedBounds.topCenter.clone();
			topCenter.y -= 20;
			const rotateHandle = createCircle(topCenter, {
				radius: 5,
				fillColor: 'yellow',
				strokeColor: 'black',
				data: { isHandle: true, corner: 'rotate' }
			});
			guideBox.addChild(rotateHandle);

			project.view.draw();
		},

		removeGuideBox() {
			if (guideBox) {
				guideBox.remove();
				guideBox = null;
			}
		},

		updateGuideBoxForSelection(groups) {
			if (!guideBox || !groups || groups.length === 0) return;

			let combinedBounds = null;
			groups.forEach((group, i) => {
				if (i === 0) {
					combinedBounds = group.bounds.clone();
				} else {
					combinedBounds = combinedBounds.unite(group.bounds);
				}
			});

			const rect = guideBox.children[0];
			rect.bounds = combinedBounds;

			const corners = [
				combinedBounds.topLeft,
				combinedBounds.topRight,
				combinedBounds.bottomRight,
				combinedBounds.bottomLeft
			];
			guideBox.children.slice(1, 5).forEach((handle, i) => {
				handle.position = corners[i];
			});

			const rotateHandle = guideBox.children[5];
			const topCenter = combinedBounds.topCenter.clone();
			topCenter.y -= 20;
			rotateHandle.position = topCenter;

			project.view.draw();
		}
	};

	/* =====================================================
        TRANSFORM OPS
    ======================================================= */
	const transformOps = {
		moveSelection(groups, event) {
			groups.forEach((group) => {
				const lastPt = group.data.lastPoint || event.point;
				const delta = event.point.subtract(lastPt);
				group.position = group.position.add(delta);
				group.data.lastPoint = event.point.clone();
			});
		},

		resizeSelection(groups, corner, event) {
			let combinedBounds = null;
			groups.forEach((group, i) => {
				if (i === 0) combinedBounds = group.bounds.clone();
				else combinedBounds = combinedBounds.unite(group.bounds);
			});

			let pivot;
			switch (corner) {
				case 0:
					pivot = combinedBounds.bottomRight;
					break;
				case 1:
					pivot = combinedBounds.bottomLeft;
					break;
				case 2:
					pivot = combinedBounds.topLeft;
					break;
				case 3:
					pivot = combinedBounds.topRight;
					break;
			}
			if (!pivot) return;

			const lastPt = groups[0].data.lastPoint || event.point;
			const delta = event.point.subtract(lastPt);

			groups.forEach((group) => {
				group.data.lastPoint = event.point.clone();
			});

			const w = combinedBounds.width;
			const h = combinedBounds.height;
			if (w < 1 || h < 1) return;

			const scaleFactor = 1 + delta.length / Math.max(w, h);
			const direction = delta.dot(event.point.subtract(pivot)) > 0;
			const finalScale = direction ? scaleFactor : 1 / scaleFactor;

			groups.forEach((group) => {
				group.scale(finalScale, pivot);
			});
		},

		rotateSelection(groups, event) {
			let combinedBounds = null;
			groups.forEach((group, i) => {
				if (i === 0) combinedBounds = group.bounds.clone();
				else combinedBounds = combinedBounds.unite(group.bounds);
			});
			const pivot = combinedBounds.center;

			const newAngle = getAngle(pivot, event.point);
			const oldAngle = groups[0].data.lastAngle ?? newAngle;
			const angleDelta = newAngle - oldAngle;

			groups.forEach((group) => {
				group.rotate(angleDelta, pivot);
				group.data.lastAngle = newAngle;
			});
		}
	};

	function getAngle(pivot, point) {
		const dx = point.x - pivot.x;
		const dy = point.y - pivot.y;
		const rad = Math.atan2(dy, dx);
		return (rad * 180) / Math.PI;
	}

	/* =====================================================
        CLONE OPS
    ======================================================= */
	function cloneGroups() {
		if (selectedGroups.length === 0) return;
		let newClones = [];
		selectedGroups.forEach((original) => {
			const cloned = original.clone();
			cloned.position = cloned.position.add(new paper.Point(10, 10));
			paper.project.activeLayer.addChild(cloned);
			initializeClonedGroupData(cloned);
			newClones.push(cloned);
		});
		selectionOps.deselectAll();
		newClones.forEach((c) =>
			selectionOps.selectGroup(c, c.children[0]?.data?.isText ? 'text' : 'shape', true)
		);
		project.view.draw();
	}

	function initializeClonedGroupData(group) {
		if (group instanceof paper.Group) {
			group.children.forEach((child) => {
				if (child instanceof paper.Path) {
					child.segments.forEach((segment) => {
						if (!segment.data) {
							segment.data = {};
						}
						// Initialize handle lengths if they are zero
						if (segment.handleIn.length === 0 && segment.handleOut.length === 0) {
							segment.handleIn = new paper.Point(-50, 0);
							segment.handleOut = new paper.Point(50, 0);
						}
					});
				} else if (child instanceof paper.PointText) {
					child.data = { isText: true };
				}
			});
		}
	}

	/* =====================================================
        TOOL INIT + EVENT HANDLERS
    ======================================================= */
	function initializePaperCanvas() {
		paper.setup(canvas);
		project = paper.project;
		mainTool = new paper.Tool();
		penTool = new paper.Tool();
		return mainTool;
	}

	function attachToolHandlers(tool) {
		function matchHitTest(hitResult) {
			const it = hitResult?.item;
			if (!it) return false;
			if (it.data?.nonInteractive && !it.data?.isHandle) return false;
			return true;
		}

		tool.onMouseDown = (event) => {
			if (isPenToolActive) return;

			const hit = project.hitTest(event.point, {
				fill: true,
				stroke: true,
				segments: true,
				tolerance: 5,
				match: matchHitTest
			});

			// If clicked empty space
			if (!hit) {
				if (isEditingShape) {
					clearAllAnchorSelections();
					project.view.draw();
				} else {
					selectionOps.deselectAll();
				}
				return;
			}

			const clickedItem = hit.item;
			const topGroup = getTopGroup(clickedItem);
			const shiftHeld = event.event.shiftKey;

			// If we are NOT in shape-edit mode, proceed with selection logic
			if (topGroup && !isEditingShape) {
				selectionOps.selectGroup(
					topGroup,
					topGroup.children[0].data.isText ? 'text' : 'shape',
					shiftHeld
				);
			}

			// If user clicked bounding box handle (rotate/resize)
			if (clickedItem?.data?.isHandle && selectedGroups.length > 0 && !isEditingShape) {
				handleTransformHandles(clickedItem, event);
			}
			// Else normal drag move
			else {
				if (selectedGroups.length > 0) {
					selectedGroups.forEach((group) => {
						group.data.lastPoint = event.point.clone();
					});
				}
			}
		};

		tool.onMouseDrag = (event) => {
			if (isPenToolActive) return;
			if (isEditingShape) return;
			if (selectedGroups.length === 0) return;

			const mode = selectedGroups[0].data.resizing;
			if ([0, 1, 2, 3].includes(mode)) {
				transformOps.resizeSelection(selectedGroups, mode, event);
			} else if (mode === 'rotate') {
				transformOps.rotateSelection(selectedGroups, event);
			} else {
				transformOps.moveSelection(selectedGroups, event);
			}
			boundingBoxOps.updateGuideBoxForSelection(selectedGroups);
		};

		tool.onMouseUp = () => {
			selectedGroups.forEach((group) => {
				group.data.resizing = null;
				group.data.lastAngle = null;
				group.data.lastPoint = null;
			});
		};
	}

	function attachPenToolHandlers(pTool) {
		let lastSegment = null;
		let lastClickTime = 0;
		let clickCount = 0;

		pTool.onMouseMove = (event) => {
			if (!isPenToolActive || !penCursor) return;
			penCursor.position = event.point;
		};

		pTool.onMouseDown = (event) => {
			if (!isPenToolActive) return;
			const now = Date.now();
			const timeSinceLast = now - lastClickTime;
			lastClickTime = now;
			clickCount = timeSinceLast < 300 ? clickCount + 1 : 1;

			if (!tempPath) {
				tempPath = new paper.Path();
				tempPath.strokeColor = 'black';
				tempPath.strokeWidth = 2;
			}
			lastSegment = tempPath.add(event.point);
			if (!lastSegment.data) {
				lastSegment.data = {};
			}
			const vert = createCircle(event.point, {
				radius: 3,
				fillColor: 'black',
				strokeColor: 'white',
				data: { nonInteractive: true }
			});
			penVertices.push(vert);
		};

		pTool.onMouseDrag = (event) => {
			if (!isPenToolActive || !tempPath || !lastSegment) return;
			const delta = event.point.subtract(lastSegment.point);
			lastSegment.handleOut = delta.multiply(0.5);
			lastSegment.handleIn = delta.multiply(-0.5);
			penCursor.position = event.point;
		};

		pTool.onMouseUp = () => {
			/* optional */
		};

		pTool.onKeyDown = (event) => {
			if (!isPenToolActive) return;
			if (event.key === 'enter' || event.key === 'escape') {
				vectorOps.finalizePath();
			}
		};
	}

	function getTopGroup(item) {
		let cur = item;
		while (cur) {
			if (cur.data?.isShapeGroup) return cur;
			cur = cur.parent;
		}
		return null;
	}

	function handleTransformHandles(handleItem, event) {
		const corner = handleItem.data.corner;
		selectedGroups.forEach((group) => {
			group.data.resizing = corner;
		});
		if (corner === 'rotate') {
			const pivot = getUnionBounds(selectedGroups).center;
			const angle = getAngle(pivot, event.point);
			selectedGroups.forEach((group) => {
				group.data.lastAngle = angle;
			});
		}
	}

	function getUnionBounds(groups) {
		let combined = null;
		groups.forEach((group, i) => {
			if (i === 0) combined = group.bounds.clone();
			else combined = combined.unite(group.bounds);
		});
		return combined;
	}

	/* =====================================================
        CLONE GROUP DATA
    ======================================================= */
	// Already handled in initializeClonedGroupData

	/* =====================================================
        INITIALIZATION
    ======================================================= */
	onMount(() => {
		mainTool = initializePaperCanvas();
		attachToolHandlers(mainTool);
		attachPenToolHandlers(penTool);
		project.view.draw();
		loadInitialItems();
	});

	function loadInitialItems() {
		if (file) {
			paper.project.importSVG(file, {
				onLoad: (item) => {
					const group = new paper.Group([item]);
					group.data = { isShapeGroup: true, editMode: false };
					paper.project.activeLayer.addChild(group);
					initializeImportedSVGData(group);
					paper.view.draw();
				},
				onError: (message) => {
					console.error('Error importing SVG:', message);
				}
			});
		}
	}

	onDestroy(() => {
		project?.clear();
		paper.view?.remove();
		mainTool?.remove();
		penTool?.remove();
	});

	/* =====================================================
        SAVE AND CANCEL FUNCTIONS
    ======================================================= */
	function doSave() {
		// selectionOps.deselectAll();
		// exitShapeEditMode();

		// Mark text items with data.isText
		paper.project.activeLayer.children.forEach((group) => {
			if (group instanceof paper.Group && group.children[0] instanceof paper.PointText) {
				const textItem = group.children[0];
				textItem.data.isText = true;
			}
		});

		const svg = paper.project.exportSVG({ asString: true });
		const thumbnailCanvas = document.createElement('canvas');
		thumbnailCanvas.width = 200;
		thumbnailCanvas.height = 150;
		const thumbnailContext = thumbnailCanvas.getContext('2d');

		const svgElement = new DOMParser().parseFromString(svg, 'image/svg+xml').documentElement;
		const svgData = new XMLSerializer().serializeToString(svgElement);
		const png = new Image();
		png.onload = () => {
			thumbnailContext.drawImage(png, 0, 0, thumbnailCanvas.width, thumbnailCanvas.height);
			const thumbnailDataUrl = thumbnailCanvas.toDataURL('image/png');
			eventDispatcher('save', { svg, png: thumbnailDataUrl });
		};
		png.src = 'data:image/svg+xml;base64,' + btoa(svgData);
	}

	function doCancel() {
		alert('Cancel button clicked!');
	}

	/* =====================================================
        REACTIVE TOP MENU UPDATE
    ======================================================= */
</script>

<Frame {width} {height} bind:verticalMenuItems bind:topMenuActions>
	<canvas
		bind:this={canvas}
		style="width: {width}; height: {height};"
		class:pen-active={isPenToolActive}
	></canvas>
</Frame>

<style>
	canvas {
		border: 1px solid #7f8c8d;
		cursor: default;
	}
	.pen-active {
		cursor: crosshair;
	}
</style>
