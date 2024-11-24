<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	import { browser } from '$app/environment';
	import { fade } from 'svelte/transition';
	import confetti from 'canvas-confetti';

	// Create a persisted store
	function createPersistedStore(key: string, initialValue: any) {
		// Load initial value from localStorage if available
		const storedValue = browser ? localStorage.getItem(key) : null;
		const initial = storedValue ? JSON.parse(storedValue) : initialValue;

		const store = writable(initial);

		// Subscribe to changes and update localStorage
		if (browser) {
			store.subscribe(value => {
				localStorage.setItem(key, JSON.stringify(value));
			});
		}

		return store;
	}

	interface Flashcard {
		id: string;
		question: string;
		answer: string;
		isFlipped?: boolean;
		showMenu?: boolean;
	}

	interface Lesson {
		id: string;
		name: string;
		created: string; // Store as ISO string instead of Date object
		archived: boolean;
		cards: Flashcard[];
		showMenu?: boolean;
	}

	const lessonsStore = createPersistedStore('flashcard-lessons', []);
	let lessons: Lesson[] = [];
	let currentLesson: Lesson | null = null;

	// Modal states
	let isNewLessonModalOpen = false;
	let isEditLessonModalOpen = false;
	let isAddCardModalOpen = false;
	let isEditCardModalOpen = false;
	let editingCard: Flashcard | null = null;
	let editQuestion = '';
	let editAnswer = '';

	// Form inputs
	let newLessonName = '';
	let newQuestion = '';
	let newAnswer = '';

	// Add new modal states for confirmations
	let isDeleteCardModalOpen = false;
	let isArchiveLessonModalOpen = false;
	let cardToDelete: Flashcard | null = null;
	let lessonToArchive: string | null = null;

	// Subscribe to store changes
	lessonsStore.subscribe(value => {
		lessons = value;
	});

	function createLesson() {
		if (newLessonName.trim()) {
			const newLesson: Lesson = {
				id: crypto.randomUUID(),
				name: newLessonName.trim(),
				created: new Date().toISOString(),
				archived: false,
				cards: []
			};
			lessonsStore.update(current => [...current, newLesson]);
			currentLesson = newLesson;
			newLessonName = '';
			closeNewLessonModal();
			
			// Add success toast
			showToastNotification('Lesson created successfully! 📚', 'success');
		}
	}

	// Add toast state
	let showToast = false;
	let toastMessage = '';
	let toastType: 'success' | 'delete' = 'success';

	function showToastNotification(message: string, type: 'success' | 'delete' = 'success') {
		toastMessage = message;
		toastType = type;
		showToast = true;
		setTimeout(() => {
			showToast = false;
		}, 3000); // Hide after 3 seconds
	}

	function addCard() {
		if (currentLesson && newQuestion.trim() && newAnswer.trim()) {
			const newCard: Flashcard = {
				id: crypto.randomUUID(),
				question: newQuestion.trim(),
				answer: newAnswer.trim()
			};
			const updatedLesson = { ...currentLesson };
			updatedLesson.cards = [...updatedLesson.cards, newCard];
			currentLesson = updatedLesson;
			
			lessonsStore.update(current => 
				current.map(l => l.id === updatedLesson.id ? updatedLesson : l)
			);
			
			newQuestion = '';
			newAnswer = '';

			// Show toast notification
			showToastNotification('Card added successfully! 🎉');

			const questionInput = document.getElementById('question');
			if (questionInput) {
				questionInput.focus();
			}
		}
	}

	function deleteCard(cardId: string) {
		if (currentLesson) {
			const updatedLesson = { ...currentLesson };
			updatedLesson.cards = updatedLesson.cards.filter(card => card.id !== cardId);
			currentLesson = updatedLesson;
			
			lessonsStore.update(current => 
				current.map(l => l.id === updatedLesson.id ? updatedLesson : l)
			);
		}
	}

	function archiveLesson(lessonId: string) {
		lessonsStore.update(current => 
			current.map(lesson => 
				lesson.id === lessonId 
					? { ...lesson, archived: true }
					: lesson
			)
		);
		
		if (currentLesson?.id === lessonId) {
			currentLesson = null;
		}
	}

	function editLesson(lesson: Lesson) {
		lessonsStore.update(current => 
			current.map(l => l.id === lesson.id ? lesson : l)
		);
	}

	function selectLesson(lesson: Lesson) {
		// First set current lesson to null to force an instant re-render
		currentLesson = null;

		// Then set the new lesson on the next tick
		setTimeout(() => {
			currentLesson = {
				...lesson,
				cards: lesson.cards.map(card => ({
					...card,
					isFlipped: false,
					showMenu: false
				}))
			};
		}, 0);
	}

	// Modal controls
	function openNewLessonModal() {
		isNewLessonModalOpen = true;
	}

	function closeNewLessonModal() {
		isNewLessonModalOpen = false;
		newLessonName = '';
	}

	function openAddCardModal() {
		isAddCardModalOpen = true;
	}

	function closeAddCardModal() {
		isAddCardModalOpen = false;
		newQuestion = '';
		newAnswer = '';
	}

	function openEditCardModal(card: Flashcard) {
		editingCard = card;
		editQuestion = card.question;
		editAnswer = card.answer;
		isEditCardModalOpen = true;
	}

	function closeEditCardModal() {
		isEditCardModalOpen = false;
		editingCard = null;
		editQuestion = '';
		editAnswer = '';
	}

	function updateCard() {
		if (currentLesson && editingCard && editQuestion.trim() && editAnswer.trim()) {
			const updatedLesson = { ...currentLesson };
			updatedLesson.cards = updatedLesson.cards.map(card => 
				card.id === editingCard?.id 
					? { ...card, question: editQuestion.trim(), answer: editAnswer.trim() }
					: card
			);
			currentLesson = updatedLesson;
			
			lessonsStore.update(current => 
				current.map(l => l.id === updatedLesson.id ? updatedLesson : l)
			);
			
			// Show toast notification
			showToastNotification('Card updated successfully! 📝');
			
			closeEditCardModal();
		}
	}

	// Add new functions for delete/archive confirmations
	function confirmDeleteCard(card: Flashcard) {
		cardToDelete = card;
		isDeleteCardModalOpen = true;
	}

	function confirmArchiveLesson(lessonId: string) {
		lessonToArchive = lessonId;
		isArchiveLessonModalOpen = true;
	}

	function executeDeleteCard() {
		if (cardToDelete && currentLesson) {
			const updatedLesson = { ...currentLesson };
			updatedLesson.cards = updatedLesson.cards.filter(card => card.id !== cardToDelete?.id);
			currentLesson = updatedLesson;
			
			lessonsStore.update(current => 
				current.map(l => l.id === updatedLesson.id ? updatedLesson : l)
			);
			
			// Show toast notification
			showToastNotification('Card deleted! 🗑️', 'delete');
			
			closeDeleteCardModal();
		}
	}

	function executeArchiveLesson() {
		if (lessonToArchive) {
			lessonsStore.update(current => 
				current.map(lesson => 
					lesson.id === lessonToArchive
						? { ...lesson, archived: true }
						: lesson
				)
			);
			
			if (currentLesson?.id === lessonToArchive) {
				currentLesson = null;
			}
			
			// Update toast message
			showToastNotification('Lesson deleted! 🗑️', 'delete');
			
			closeArchiveLessonModal();
		}
	}

	function closeDeleteCardModal() {
		isDeleteCardModalOpen = false;
		cardToDelete = null;
	}

	function closeArchiveLessonModal() {
		isArchiveLessonModalOpen = false;
		lessonToArchive = null;
	}

	// Add these state variables
	let editLessonName = '';
	let editingLesson: Lesson | null = null;

	// Add these functions
	function openEditLessonModal(lesson: Lesson) {
		editingLesson = lesson;
		editLessonName = lesson.name;
		isEditLessonModalOpen = true;
	}

	function closeEditLessonModal() {
		isEditLessonModalOpen = false;
		editingLesson = null;
		editLessonName = '';
	}

	function updateLesson() {
		if (editingLesson && editLessonName.trim()) {
			const updatedLesson = {
				...editingLesson,
				name: editLessonName.trim()
			};
			
			lessonsStore.update(current => 
				current.map(l => l.id === editingLesson.id ? updatedLesson : l)
			);
			
			if (currentLesson?.id === editingLesson.id) {
				currentLesson = updatedLesson;
			}
			
			closeEditLessonModal();
			
			// Add success toast
			showToastNotification('Lesson updated successfully! 📝', 'success');
		}
	}

	// Add theme management
	let isDarkMode = false;

	// Load theme preference on mount
	onMount(() => {
		if (browser) {
			// Check local storage first
			const storedTheme = localStorage.getItem('theme');
			if (storedTheme) {
				isDarkMode = storedTheme === 'dark';
			} else {
				// Check system preference
				isDarkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;
			}
			updateTheme(isDarkMode);
		}
	});

	function toggleTheme() {
		isDarkMode = !isDarkMode;
		updateTheme(isDarkMode);
		if (browser) {
			localStorage.setItem('theme', isDarkMode ? 'dark' : 'light');
		}
	}

	function updateTheme(dark: boolean) {
		if (browser) {
			document.documentElement.classList.toggle('dark-theme', dark);
			document.body.classList.toggle('dark-theme', dark);
		}
	}

	// Add quiz-related state
	let isQuizModalOpen = false;
	let quizCards: (Flashcard & { isCorrect?: boolean })[] = [];
	let currentQuizIndex = 0;
	let quizIsComplete = false;
	let quizScore = 0;
	let currentCardFlipped = false;

	// Add new state for quiz content visibility
	let showQuizContent = true;

	// Add state for reverse quiz
	let isReverseQuiz = false;

	function startQuiz(reverse: boolean = false) {
		if (currentLesson) {
			isReverseQuiz = reverse;
			// Shuffle the cards
			quizCards = [...currentLesson.cards]
				.sort(() => Math.random() - 0.5)
				.map(card => ({ ...card }));
			currentQuizIndex = 0;
			quizIsComplete = false;
			quizScore = 0;
			currentCardFlipped = false;
			showQuizContent = true;
			isQuizModalOpen = true;
		}
	}

	function handleAnswer(isCorrect: boolean) {
		// First, hide all content with quick fade
		showQuizContent = false;

		// Record the answer for the current card
		quizCards = quizCards.map((card, index) => 
			index === currentQuizIndex 
				? { ...card, isCorrect: isCorrect }
				: card
		);

		// Wait for fade out to complete
		setTimeout(() => {
			if (currentQuizIndex < quizCards.length - 1) {
				currentCardFlipped = false;
				currentQuizIndex++;
				
				// Wait for card to be in position, then fade in content
				setTimeout(() => {
					showQuizContent = true;
				}, 150);  // Match the CSS transition time
			} else {
				// Quiz is complete
				quizIsComplete = true;
				quizScore = quizCards.filter(card => card.isCorrect).length;
				
				if (quizScore === quizCards.length) {
					fireConfetti();
				}
			}
		}, 150);  // Match the CSS transition time
	}

	function flipQuizCard() {
		if (!currentCardFlipped) {
			showQuizContent = false;
			setTimeout(() => {
				currentCardFlipped = true;
				setTimeout(() => {
					showQuizContent = true;
				}, 150);  // Match the CSS transition time
			}, 150);  // Match the CSS transition time
		}
	}

	function closeQuizModal() {
		isQuizModalOpen = false;
		quizCards = [];
		currentQuizIndex = 0;
			quizIsComplete = false;
		currentCardFlipped = false;
	}

	function fireConfetti() {
		const count = 200;
		const defaults = {
			origin: { y: 0.7 },
			zIndex: 1500
		};

		function fire(particleRatio: number, opts: any) {
			confetti({
				...defaults,
				...opts,
				particleCount: Math.floor(count * particleRatio)
			});
		}

		fire(0.25, {
			spread: 26,
			startVelocity: 55,
		});

		fire(0.2, {
			spread: 60,
		});

		fire(0.35, {
			spread: 100,
			decay: 0.91,
			scalar: 0.8
		});

		fire(0.1, {
			spread: 120,
			startVelocity: 25,
			decay: 0.92,
			scalar: 1.2
		});

		fire(0.1, {
			spread: 120,
			startVelocity: 45,
		});
	}

	function flipAllCards() {
		if (currentLesson) {
			const allFlipped = currentLesson.cards.every(card => card.isFlipped);
			const updatedLesson = {
				...currentLesson,
				cards: currentLesson.cards.map(card => ({
					...card,
					isFlipped: !allFlipped
				}))
			};
			currentLesson = updatedLesson;
			lessonsStore.update(current => 
				current.map(l => l.id === updatedLesson.id ? updatedLesson : l)
			);
		}
	}
</script>

<main class="container">
	<div class="header">
		<div class="logo">
			Flash<span class="logo-accent">Dash</span>
			<div class="logo-dot"></div>
		</div>
		<div class="header-controls">
			<button class="secondary-button" on:click={toggleTheme} title="Toggle theme">
				{#if isDarkMode}
					<span class="icon">☀️</span> Light Mode
				{:else}
					<span class="icon">🌙</span> Dark Mode
				{/if}
			</button>
		</div>
	</div>

	<div class="lessons-container">
		<div class="lessons-sidebar">
			<div class="sidebar-header">
				<h2>Your Lessons</h2>
				<button class="circle-button" on:click={openNewLessonModal} title="Create New Lesson">
					<span class="plus-icon">+</span>
				</button>
			</div>
			<div class="lessons-list">
				{#each lessons.filter(l => !l.archived) as lesson}
					<div 
						class="lesson-item" 
						class:active={currentLesson?.id === lesson.id}
						on:click={() => selectLesson(lesson)}
					>
						<span class="lesson-name">{lesson.name}</span>
						<div class="lesson-actions">
							<div class="dropdown">
								<button 
									class="action-button menu-button" 
									on:click|stopPropagation={() => lesson.showMenu = !lesson.showMenu}
									title="Lesson Options"
								>
									⋮
								</button>
								{#if lesson.showMenu}
									<div 
										class="dropdown-menu" 
										on:click|stopPropagation
									>
										<button 
											class="dropdown-item"
											on:click={() => {
												lesson.showMenu = false;
												openEditLessonModal(lesson);
											}}
										>
											<span class="dropdown-icon">✏️</span>
											Edit
										</button>
										<button 
											class="dropdown-item delete"
											on:click={() => {
												lesson.showMenu = false;
												confirmArchiveLesson(lesson.id);
											}}
										>
											<span class="dropdown-icon">🗑</span>
											Delete
										</button>
									</div>
								{/if}
							</div>
						</div>
					</div>
				{/each}
			</div>
		</div>

		<div class="lesson-content">
			{#if currentLesson}
				<div class="lesson-header">
					<div class="lesson-title-row">
						<div class="lesson-controls">
							{#if currentLesson.cards.length > 0}
								<button class="primary-button" on:click={startQuiz}>
									<span class="icon">📝</span> Quiz
								</button>
								<button class="primary-button" on:click={() => startQuiz(true)}>
									<span class="icon">🔄</span> Reverse Quiz
								</button>
							{/if}
							<button class="secondary-button" on:click={() => flipAllCards()}>
								<span class="icon">🔄</span> Flip All
							</button>
							<button class="primary-button" on:click={openAddCardModal}>
								<span class="icon">➕</span> Add Card
							</button>
						</div>
					</div>
				</div>

				{#if currentLesson.cards.length > 0}
					<div class="cards-grid">
						{#each currentLesson.cards as card}
							<div class="grid-card" class:flipped={card.isFlipped} on:click={() => card.isFlipped = !card.isFlipped}>
								<div class="grid-card-inner">
									<div class="grid-card-front">
										<div class="card-label">Q</div>
										<p>{card.question}</p>
										<div class="card-actions">
											<div class="dropdown">
												<button 
													class="action-button menu-button" 
													on:click|stopPropagation={() => card.showMenu = !card.showMenu}
													title="Card Options"
												>
													⋮
												</button>
												{#if card.showMenu}
													<div class="dropdown-menu" on:click|stopPropagation>
														<button 
															class="dropdown-item"
															on:click={() => {
																card.showMenu = false;
																openEditCardModal(card);
															}}
														>
															<span class="dropdown-icon">✏️</span>
															Edit
														</button>
														<button 
															class="dropdown-item delete"
															on:click={() => {
																card.showMenu = false;
																confirmDeleteCard(card);
															}}
														>
															<span class="dropdown-icon">🗑</span>
															Delete
														</button>
													</div>
												{/if}
											</div>
										</div>
									</div>
									<div class="grid-card-back">
										<div class="card-label">A</div>
										<p>{card.answer}</p>
										<div class="card-actions">
											<div class="dropdown">
												<button 
													class="action-button menu-button" 
													on:click|stopPropagation={() => card.showMenu = !card.showMenu}
													title="Card Options"
												>
													⋮
												</button>
												{#if card.showMenu}
													<div class="dropdown-menu" on:click|stopPropagation>
														<button 
															class="dropdown-item"
															on:click={() => {
																card.showMenu = false;
																openEditCardModal(card);
															}}
														>
															<span class="dropdown-icon">✏️</span>
															Edit
														</button>
														<button 
															class="dropdown-item delete"
															on:click={() => {
																card.showMenu = false;
																confirmDeleteCard(card);
															}}
														>
															<span class="dropdown-icon">🗑</span>
															Delete
														</button>
													</div>
												{/if}
											</div>
										</div>
									</div>
								</div>
							</div>
						{/each}
					</div>
				{:else}
					<div class="empty-state">
						<div class="empty-state-content">
							<p>No flashcards yet. Add some to get started!</p>
							<button class="primary-button" on:click={openAddCardModal}>
								<span class="icon">➕</span> Add Your First Card
							</button>
						</div>
					</div>
				{/if}
			{:else}
				<div class="empty-state">
					<div class="empty-state-content">
						<p>Select a lesson from the sidebar or create a new one to get started!</p>
						<button class="primary-button" on:click={openNewLessonModal}>
							<span class="icon"></span> Create New Lesson
						</button>
					</div>
				</div>
			{/if}
		</div>
	</div>
</main>

<!-- New Lesson Modal -->
{#if isNewLessonModalOpen}
	<div class="slide-overlay" on:click={closeNewLessonModal}>
		<div class="slide-panel" on:click|stopPropagation>
			<div class="slide-header">
				<h2>Create New Lesson</h2>
				<button class="close-button" on:click={closeNewLessonModal}>×</button>
			</div>
			<div class="slide-content">
				<div class="input-group">
					<label for="lessonName">Lesson Name:</label>
					<input
						id="lessonName"
						type="text"
						placeholder="Enter lesson name"
						bind:value={newLessonName}
					/>
				</div>
			</div>
			<div class="slide-footer">
				<button class="secondary-button" on:click={closeNewLessonModal}>Cancel</button>
				<button class="primary-button" on:click={createLesson}>Create Lesson</button>
			</div>
		</div>
	</div>
{/if}

<!-- Add Card Slide-out -->
{#if isAddCardModalOpen}
	<div class="slide-overlay" on:click={closeAddCardModal}>
		<div class="slide-panel" on:click|stopPropagation>
			<div class="slide-header">
				<h2>Add New Card</h2>
				<button class="close-button" on:click={closeAddCardModal}>×</button>
			</div>
			<div class="slide-content">
				<div class="input-group">
					<label for="question">Question:</label>
					<textarea
						id="question"
						rows="4"
						placeholder="Enter question"
						bind:value={newQuestion}
					/>
				</div>
				<div class="input-group">
					<label for="answer">Answer:</label>
					<textarea
						id="answer"
						rows="4"
						placeholder="Enter answer"
						bind:value={newAnswer}
					/>
				</div>
			</div>
			<div class="slide-footer">
				<button class="secondary-button" on:click={closeAddCardModal}>Cancel</button>
				<button class="primary-button" on:click={addCard}>Add Card</button>
			</div>
		</div>
	</div>
{/if}

<!-- Edit Card Slide-out -->
{#if isEditCardModalOpen}
	<div class="slide-overlay" on:click={closeEditCardModal}>
		<div class="slide-panel" on:click|stopPropagation>
			<div class="slide-header">
				<h2>Edit Card</h2>
				<button class="close-button" on:click={closeEditCardModal}>×</button>
			</div>
			<div class="slide-content">
				<div class="input-group">
					<label for="editQuestion">Question:</label>
					<textarea
						id="editQuestion"
						rows="4"
						placeholder="Enter question"
						bind:value={editQuestion}
					/>
				</div>
				<div class="input-group">
					<label for="editAnswer">Answer:</label>
					<textarea
						id="editAnswer"
						rows="4"
						placeholder="Enter answer"
						bind:value={editAnswer}
					/>
				</div>
			</div>
			<div class="slide-footer">
				<button class="secondary-button" on:click={closeEditCardModal}>Cancel</button>
				<button class="primary-button" on:click={updateCard}>Save Changes</button>
			</div>
		</div>
	</div>
{/if}

<!-- Delete Card Confirmation Modal -->
{#if isDeleteCardModalOpen}
	<div class="modal-overlay" on:click={closeDeleteCardModal}>
		<div class="modal confirmation-modal" on:click|stopPropagation>
			<h2>Delete Card</h2>
			<p class="confirmation-message">Are you sure you want to delete this card? This action cannot be undone.</p>
			<div class="card-preview">
				<div class="preview-section">
					<strong>Question:</strong>
					<p>{cardToDelete?.question}</p>
				</div>
				<div class="preview-section">
					<strong>Answer:</strong>
					<p>{cardToDelete?.answer}</p>
				</div>
			</div>
			<div class="modal-buttons">
				<button class="secondary-button" on:click={closeDeleteCardModal}>Cancel</button>
				<button class="danger-button" on:click={executeDeleteCard}>Delete Card</button>
			</div>
		</div>
	</div>
{/if}

<!-- Archive Lesson Confirmation Modal -->
{#if isArchiveLessonModalOpen}
	<div class="modal-overlay" on:click={closeArchiveLessonModal}>
		<div class="modal confirmation-modal" on:click|stopPropagation>
			<h2>Delete Lesson</h2>
			<p class="confirmation-message">
				Are you sure you want to delete this lesson? This will remove the lesson and all its cards.
				This action cannot be undone.
			</p>
			<div class="modal-buttons">
				<button class="secondary-button" on:click={closeArchiveLessonModal}>Cancel</button>
				<button class="danger-button" on:click={executeArchiveLesson}>Delete Lesson</button>
			</div>
		</div>
	</div>
{/if}

<!-- Edit Lesson Modal -->
{#if isEditLessonModalOpen}
	<div class="slide-overlay" on:click={closeEditLessonModal}>
		<div class="slide-panel" on:click|stopPropagation>
			<div class="slide-header">
				<h2>Edit Lesson</h2>
				<button class="close-button" on:click={closeEditLessonModal}>×</button>
			</div>
			<div class="slide-content">
				<div class="input-group">
					<label for="editLessonName">Lesson Name:</label>
					<input
						id="editLessonName"
						type="text"
						placeholder="Enter lesson name"
						bind:value={editLessonName}
					/>
				</div>
			</div>
			<div class="slide-footer">
				<button class="secondary-button" on:click={closeEditLessonModal}>Cancel</button>
				<button class="primary-button" on:click={updateLesson}>Save Changes</button>
			</div>
		</div>
	</div>
{/if}

<!-- Add the Quiz Modal -->
{#if isQuizModalOpen}
	<div class="modal-overlay quiz-modal-overlay" on:click={closeQuizModal}>
		<div class="modal quiz-modal" on:click|stopPropagation>
			{#if !quizIsComplete}
				<div class="quiz-header">
					<h2>Quiz Mode</h2>
					<div class="quiz-progress">
						Card {currentQuizIndex + 1} of {quizCards.length}
					</div>
				</div>
				<div class="quiz-card" class:flipped={currentCardFlipped}>
					<div class="quiz-card-inner">
						{#if isReverseQuiz}
							<!-- Reverse Quiz Template -->
							<div class="quiz-card-front" on:click={flipQuizCard}>
								<div class="card-label">A</div>
								<div class="quiz-content" class:hidden={!showQuizContent}>
									<p>{quizCards[currentQuizIndex]?.answer}</p>
									<div class="quiz-hint">Click to reveal question</div>
								</div>
							</div>
							<div class="quiz-card-back">
								<div class="card-label">Q</div>
								<div class="quiz-content" class:hidden={!showQuizContent}>
									<p>{quizCards[currentQuizIndex]?.question}</p>
									<div class="quiz-actions">
										<button class="quiz-button incorrect" on:click={() => handleAnswer(false)}>
											❌ Incorrect
										</button>
										<button class="quiz-button correct" on:click={() => handleAnswer(true)}>
											✅ Correct
										</button>
									</div>
								</div>
							</div>
						{:else}
							<!-- Regular Quiz Template (unchanged) -->
							<div class="quiz-card-front" on:click={flipQuizCard}>
								<div class="card-label">Q</div>
								<div class="quiz-content" class:hidden={!showQuizContent}>
									<p>{quizCards[currentQuizIndex]?.question}</p>
									<div class="quiz-hint">Click to reveal answer</div>
								</div>
							</div>
							<div class="quiz-card-back">
								<div class="card-label">A</div>
								<div class="quiz-content" class:hidden={!showQuizContent}>
									<p>{quizCards[currentQuizIndex]?.answer}</p>
									<div class="quiz-actions">
										<button class="quiz-button incorrect" on:click={() => handleAnswer(false)}>
											❌ Incorrect
										</button>
										<button class="quiz-button correct" on:click={() => handleAnswer(true)}>
											✅ Correct
										</button>
									</div>
								</div>
							</div>
						{/if}
					</div>
				</div>
			{:else}
				<div class="quiz-results">
					<h2>Quiz Complete! 🎉</h2>
					<div class="score-display">
						<div class="score">
							{quizScore} / {quizCards.length}
						</div>
						<div class="percentage">
							{Math.round((quizScore / quizCards.length) * 100)}%
						</div>
					</div>
					<div class="results-breakdown">
						<h3>Breakdown:</h3>
						<div class="results-list">
							{#each quizCards as card}
								<div class="result-item" class:correct={card.isCorrect}>
									<span class="result-icon">{card.isCorrect ? '✅' : '❌'}</span>
									<div class="result-content">
										<div class="result-question">{card.question}</div>
										<div class="result-answer">{card.answer}</div>
									</div>
								</div>
							{/each}
						</div>
					</div>
					<div class="quiz-footer">
						<button class="secondary-button" on:click={closeQuizModal}>Close</button>
						<button class="primary-button" on:click={startQuiz}>Try Again</button>
					</div>
				</div>
			{/if}
		</div>
	</div>
{/if}

<footer class="footer">
	Made with <span class="heart">❤️</span> for the Malkos
</footer>

{#if showToast}
	<div class="toast" class:toast-delete={toastType === 'delete'} transition:fade>
		{toastMessage}
	</div>
{/if}

<style>
	:global(:root) {
		--bg-primary: #fafafa;
		--bg-secondary: #ffffff;
		--bg-tertiary: #f5f5f5;
		--text-primary: #111827;
		--text-secondary: #374151;
		--text-tertiary: #6b7280;
		--border-color: #e5e7eb;
		--primary-color: #3b82f6;
		--primary-hover: #2563eb;
		--danger-color: #ef4444;
		--danger-hover: #dc2626;
		--card-shadow: 0 1px 3px rgba(0, 0, 0, 0.1), 0 1px 2px -1px rgba(0, 0, 0, 0.1);
		--success-color: #10b981;
	}

	:global(.dark-theme) {
		--bg-primary: #0f172a;
		--bg-secondary: #1e293b;
		--bg-tertiary: #334155;
		--text-primary: #f8fafc;
		--text-secondary: #cbd5e1;
		--text-tertiary: #94a3b8;
		--border-color: #475569;
		--primary-color: #3b82f6;
		--primary-hover: #60a5fa;
		--danger-color: #fbbf24;
		--danger-hover: #fcd34d;
		--card-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.3), 0 2px 4px -2px rgba(0, 0, 0, 0.2);
		--success-color: #4ade80;
		--border-hover: #64748b;
	}

	.container {
		max-width: 1280px;
		margin: 0 auto;
		padding: 1.5rem;
			min-height: calc(100vh - 3rem);
	}

	.lessons-container {
		display: grid;
		grid-template-columns: 300px 1fr;
		gap: 0;
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		border-radius: 16px;
		box-shadow: var(--card-shadow);
		overflow: hidden;
		margin-top: 2rem;
		min-height: calc(100vh - 8rem);
	}

	.lessons-sidebar {
		background: var(--bg-primary);
		border-right: 1px solid var(--border-color);
		padding: 1.25rem;
	}

	.lessons-list {
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
	}

	.lesson-content {
		padding: 1.5rem;
		overflow-y: auto;
	}

	.cards-grid {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
		gap: 1.25rem;
		padding: 0.5rem;
	}

	.lesson-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		padding: 0.875rem 1rem;
		border-radius: 8px;
		font-weight: 500;
		color: var(--text-primary);
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.lesson-item:hover {
		background: var(--bg-tertiary);
		border-color: var(--border-color);
		transform: translateY(-1px);
	}

	.lesson-item.active {
		background: var(--primary-color);
		border-color: var(--primary-color);
		color: white;
	}

	.grid-card {
		height: 220px;
		perspective: 1000px;
		cursor: pointer;
	}

	.grid-card-inner {
		position: relative;
		width: 100%;
		height: 100%;
		text-align: center;
		transition: transform 0.6s;
		transform-style: preserve-3d;
	}

	.grid-card.flipped .grid-card-inner {
		transform: rotateY(180deg);
	}

	.grid-card-front,
	.grid-card-back {
		position: absolute;
		width: 100%;
		height: 100%;
		-webkit-backface-visibility: hidden;
		backface-visibility: hidden;
		display: flex;
		align-items: center;
		justify-content: center;
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		border-radius: 12px;
		box-shadow: var(--card-shadow);
		padding: 1.5rem;
	}

	.grid-card-back {
		transform: rotateY(180deg);
		background: var(--bg-secondary);
	}

	.card-label {
		position: absolute;
		top: 1rem;
		left: 1rem;
		width: 28px;
		height: 28px;
		background-color: var(--primary-color);
		color: white;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-weight: 600;
		font-size: 0.875rem;
	}

	.grid-card-back .card-label {
		background-color: var(--success-color);
	}

	.card-actions {
		position: absolute;
		top: 0.75rem;
		right: 0.75rem;
		z-index: 5;
	}

	.grid-card p {
		margin: 0;
		padding: 1rem;
		width: 100%;
		height: 100%;
		font-size: 1rem;
		line-height: 1.5;
		color: var(--text-primary);
		overflow-y: auto;
		scrollbar-width: thin;
		scrollbar-color: var(--primary-color) transparent;
	}

	.grid-card p::-webkit-scrollbar {
		width: 6px;
	}

	.grid-card p::-webkit-scrollbar-track {
		background: transparent;
	}

	.grid-card p::-webkit-scrollbar-thumb {
		background-color: var(--primary-color);
		border-radius: 3px;
	}

	.grid-card-front p,
	.grid-card-back p {
		padding-top: 3rem;
		padding-left: 1rem;
		padding-right: 1rem;
		padding-bottom: 1rem;
	}

	.primary-button {
		background-color: var(--primary-color);
		color: white;
		padding: 0.75rem 1.5rem;
		border: none;
		border-radius: 8px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		display: inline-flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 0.875rem;
		box-shadow: 0 2px 4px rgba(59, 130, 246, 0.2);
	}

	.primary-button:hover {
		background-color: var(--primary-hover);
		transform: translateY(-1px);
		box-shadow: 0 4px 8px rgba(59, 130, 246, 0.3);
	}

	.secondary-button {
		background-color: var(--bg-tertiary);
		color: var(--text-primary);
		padding: 0.75rem 1.5rem;
		border: 1px solid var(--border-color);
		border-radius: 8px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		display: inline-flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 0.875rem;
	}

	.secondary-button:hover {
		background-color: var(--bg-secondary);
		border-color: var(--primary-color);
		color: var(--primary-color);
		transform: translateY(-1px);
	}

	.danger-button {
		background-color: var(--danger-color);
		color: white;
		padding: 0.75rem 1.5rem;
		border: none;
		border-radius: 8px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		display: inline-flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 0.875rem;
	}

	.danger-button:hover {
		background-color: var(--danger-hover);
		transform: translateY(-1px);
	}

	.empty-state-content {
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		padding: 3rem;
		border-radius: 16px;
	}

	.empty-state-content p {
		font-weight: 500;
		font-size: 1.125rem;
		margin-bottom: 2rem;
	}

	.slide-panel {
		background: var(--bg-secondary);
	}

	.slide-header {
		padding: 1.5rem;
		border-bottom: 1px solid var(--border-color);
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.slide-header h2 {
		font-size: 1.25rem;
		font-weight: 600;
		color: var(--text-primary);
		margin: 0;
		letter-spacing: -0.01em;
	}

	.input-group label {
		font-weight: 600;
		font-size: 0.875rem;
		color: var(--text-primary);
	}

	.input-group input,
	.input-group textarea {
		width: 100%;
		padding: 0.75rem;
		border: 1px solid var(--border-color);
		border-radius: 8px;
		font-size: 0.9375rem;
		background: var(--bg-primary);
		color: var(--text-primary);
		transition: all 0.2s ease;
	}

	.input-group input:focus,
	.input-group textarea:focus {
		outline: none;
		border-color: var(--primary-color);
		box-shadow: 0 0 0 2px rgba(96, 165, 250, 0.2);
	}

	.dropdown-menu {
		position: absolute;
		top: 100%;
		right: 0;
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		border-radius: 8px;
		box-shadow: var(--card-shadow);
		min-width: 140px;
		z-index: 10;
		padding: 0.5rem;
	}

	.dropdown-item {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		width: 100%;
		padding: 0.625rem 0.75rem;
		border: none;
		background: none;
		text-align: left;
		cursor: pointer;
		color: var(--text-primary);
		transition: all 0.2s;
		border-radius: 4px;
		font-size: 0.875rem;
		font-weight: 500;
	}

	.dropdown-item:hover {
		background-color: var(--bg-tertiary);
	}

	.dropdown-item.delete {
		color: var(--danger-color);
	}

	.dropdown-item.delete:hover {
		background-color: var(--danger-color);
		color: white;
	}

	.lesson-name {
		color: inherit;
	}

	.logo {
		font-size: 2rem;
		font-weight: 800;
		letter-spacing: -0.03em;
		position: relative;
		display: flex;
			align-items: center;
		font-family: -apple-system, BlinkMacSystemFont, 'Inter', 'Segoe UI', Roboto, sans-serif;
		background: linear-gradient(135deg, var(--text-primary) 60%, var(--primary-color));
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
	}

	.logo-accent {
		background: linear-gradient(135deg, var(--primary-color), #60a5fa);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
	}

	.logo-dot {
		width: 8px;
		height: 8px;
		background: var(--primary-color);
		border-radius: 50%;
		margin-left: 4px;
		animation: pulse 2s infinite;
	}

	@keyframes pulse {
		0% {
			transform: scale(1);
			opacity: 1;
		}
		50% {
			transform: scale(1.2);
			opacity: 0.8;
		}
		100% {
			transform: scale(1);
			opacity: 1;
		}
	}

	.empty-state {
		height: 100%;
		min-height: 400px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.empty-state-content {
		text-align: center;
		max-width: 400px;
		padding: 2rem;
		background: var(--bg-secondary);
		border-radius: 12px;
		box-shadow: var(--card-shadow);
	}

	.empty-state-content p {
		margin-bottom: 1.5rem;
		color: var(--text-secondary);
		font-size: 1.1rem;
		line-height: 1.5;
	}

	.empty-state-content .primary-button {
		margin: 0 auto;
		padding: 1rem 2rem;
		font-size: 1rem;
	}

	.circle-button {
		width: 40px;
		height: 40px;
		border-radius: 50%;
		border: none;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		transition: all 0.2s ease;
		padding: 0;
		background-color: var(--primary-color);
		color: white;
		font-size: 1.5rem;
		box-shadow: var(--card-shadow);
	}

	.circle-button:hover {
		background-color: var(--primary-hover);
		transform: scale(1.05);
	}

	.menu-button {
		color: var(--text-secondary);
		background: transparent;
		border: none;
		width: 32px;
		height: 32px;
		display: flex;
		align-items: center;
		justify-content: center;
		border-radius: 6px;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.menu-button:hover {
		background-color: var(--bg-tertiary);
		color: var(--text-primary);
	}

	.slide-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0, 0, 0, 0.3);
		display: flex;
		justify-content: flex-end;
		z-index: 1000;
		animation: fadeIn 0.3s ease;
	}

	.slide-panel {
		width: 100%;
		max-width: 500px;
		background: var(--bg-secondary);
		height: 100%;
		display: flex;
		flex-direction: column;
		animation: slideIn 0.3s ease;
		box-shadow: -4px 0 25px rgba(0, 0, 0, 0.15);
	}

	.slide-content {
		flex: 1;
		padding: 1.5rem;
		overflow-y: auto;
	}

	.slide-footer {
		padding: 1.5rem;
		border-top: 1px solid var(--border-color);
		display: flex;
		justify-content: flex-end;
		gap: 1rem;
		background: var(--bg-secondary);
		box-shadow: 0 -4px 6px -1px rgba(0, 0, 0, 0.05);
	}

	.modal-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: var(--bg-primary);
		background-image: radial-gradient(var(--bg-secondary) 1px, transparent 1px);
		background-size: 20px 20px;
		backdrop-filter: blur(4px);
		display: flex;
		justify-content: center;
		align-items: center;
		z-index: 1000;
	}

	.modal {
		background: var(--bg-secondary);
		padding: 2rem;
		border-radius: 12px;
		width: 90%;
		max-width: 500px;
		box-shadow: var(--card-shadow);
	}

	.confirmation-modal {
		max-width: 400px;
	}

	.close-button {
		background: none;
		border: none;
		font-size: 1.5rem;
		color: var(--text-secondary);
			cursor: pointer;
		padding: 0.5rem;
		margin: -0.5rem;
		line-height: 1;
		transition: all 0.2s ease;
		border-radius: 8px;
		width: 40px;
		height: 40px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.close-button:hover {
		background-color: var(--bg-tertiary);
		color: var(--text-primary);
		transform: scale(1.05);
	}

	.confirmation-message {
		color: var(--text-secondary);
		margin-bottom: 1.5rem;
		line-height: 1.5;
	}

	.card-preview {
		background: var(--bg-tertiary);
		border: 1px solid var(--border-color);
		border-radius: 8px;
		padding: 1.5rem;
		margin-bottom: 2rem;
	}

	.preview-section {
		margin-bottom: 1.5rem;
	}

	.preview-section:last-child {
		margin-bottom: 0;
	}

	.preview-section strong {
		display: block;
		margin-bottom: 0.75rem;
		color: var(--text-primary);
		font-size: 0.875rem;
		text-transform: uppercase;
		letter-spacing: 0.025em;
	}

	.preview-section p {
		margin: 0;
		color: var(--text-secondary);
		line-height: 1.5;
		font-size: 0.9375rem;
	}

	@keyframes slideIn {
		from {
			transform: translateX(100%);
		}
		to {
			transform: translateX(0);
		}
	}

	@keyframes fadeIn {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}

	.header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 2rem;
		padding: 0.5rem 0;
	}

	.header-controls {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.lessons-sidebar h2 {
		font-size: 1.25rem;
		font-weight: 600;
		color: var(--text-primary);
		margin: 0 0 1.25rem 0;
		padding: 0.5rem;
		letter-spacing: -0.01em;
		border-bottom: 2px solid var(--border-color);
		position: relative;
	}

	.lessons-sidebar h2::after {
		content: '';
		position: absolute;
		bottom: -2px;
		left: 0;
		width: 60px;
		height: 2px;
		background: linear-gradient(to right, var(--primary-color), transparent);
		transition: width 0.3s ease;
	}

	.lessons-sidebar h2:hover::after {
		width: 100%;
	}

	.lesson-content h2 {
		font-size: 1.5rem;
		font-weight: 600;
		color: var(--text-primary);
		margin: 0;
		letter-spacing: -0.01em;
	}

	.lesson-title-row {
		display: flex;
		justify-content: flex-end;
		padding: 0.75rem 0.5rem;
		margin-bottom: 2rem;
		border-bottom: 2px solid var(--border-color);
		position: relative;
	}

	.lesson-title-row::after {
		content: '';
		position: absolute;
		bottom: -2px;
		left: 0;
		width: 100px;
		height: 2px;
		background: linear-gradient(to right, var(--primary-color), transparent);
	}

	.lessons-sidebar {
		background: var(--bg-primary);
		border-right: 1px solid var(--border-color);
		padding: 1.5rem;
	}

	.lesson-content {
		padding: 1.5rem 2rem;
		overflow-y: auto;
	}

	.sidebar-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-bottom: 1.25rem;
		padding: 0.5rem;
		border-bottom: 2px solid var(--border-color);
		position: relative;
	}

	.sidebar-header h2 {
		font-size: 1.25rem;
		font-weight: 600;
		color: var(--text-primary);
		margin: 0;
		padding: 0;
		letter-spacing: -0.01em;
		border-bottom: none;
	}

	.sidebar-header::after {
		content: '';
		position: absolute;
		bottom: -2px;
		left: 0;
		width: 60px;
		height: 2px;
		background: linear-gradient(to right, var(--primary-color), transparent);
		transition: width 0.3s ease;
	}

	.sidebar-header:hover::after {
		width: 100%;
	}

	.circle-button {
		width: 32px;
		height: 32px;
		min-width: 32px;
		font-size: 1.25rem;
	}

	.lessons-sidebar h2 {
		border-bottom: none;
		margin: 0;
		padding: 0;
	}

	.lessons-sidebar h2::after {
		display: none;
	}

	.footer {
		text-align: center;
		padding: 1rem 0;
		color: var(--text-secondary);
		font-size: 0.875rem;
		margin-top: 2rem;
	}

	.heart {
		display: inline-block;
		animation: heartbeat 1.5s ease infinite;
	}

	@keyframes heartbeat {
		0% {
			transform: scale(1);
		}
		50% {
			transform: scale(1.1);
		}
		100% {
			transform: scale(1);
		}
	}

	.toast {
		position: fixed;
		bottom: 2rem;
		left: 50%;
		transform: translateX(-50%);
		background-color: #064e3b; /* Dark green for success */
		color: white;
		padding: 1rem 2rem;
		border-radius: 8px;
		font-weight: 500;
		box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.2);
		z-index: 1000;
		animation: slideUp 0.3s ease;
	}

	.toast-delete {
		background-color: #dc2626; /* Red for delete */
		border: 1px solid #ef4444;
		box-shadow: 0 4px 6px -1px rgba(220, 38, 38, 0.2);
	}

	@keyframes slideUp {
		from {
			transform: translate(-50%, 100%);
			opacity: 0;
		}
		to {
						transform: translate(-50%, 0);
						opacity: 1;
					}
	}

	.modal h2 {
		color: var(--text-primary);
		margin: 0 0 1.5rem 0;
		font-size: 1.25rem;
		font-weight: 600;
		letter-spacing: -0.01em;
	}

	.confirmation-modal {
		max-width: 400px;
		padding: 2rem;
	}

	.confirmation-message {
		color: var(--text-secondary);
		margin-bottom: 2rem;
		line-height: 1.5;
		font-size: 0.9375rem;
	}

	.card-preview {
		background: var(--bg-tertiary);
		border: 1px solid var(--border-color);
		border-radius: 8px;
		padding: 1.5rem;
		margin-bottom: 2rem;
	}

	.preview-section {
		margin-bottom: 1.5rem;
	}

	.preview-section:last-child {
		margin-bottom: 0;
	}

	.preview-section strong {
		display: block;
		margin-bottom: 0.75rem;
		color: var(--text-primary);
		font-size: 0.875rem;
		text-transform: uppercase;
		letter-spacing: 0.025em;
	}

	.preview-section p {
		margin: 0;
		color: var(--text-secondary);
		line-height: 1.5;
		font-size: 0.9375rem;
	}

	.quiz-modal {
		max-width: 600px;
		padding: 0;
		display: flex;
		flex-direction: column;
	}

	.quiz-header {
		padding: 1.5rem;
		border-bottom: 1px solid var(--border-color);
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.quiz-progress {
		color: var(--text-secondary);
		font-size: 0.875rem;
	}

	.quiz-card {
		height: 400px;
		perspective: 1000px;
		padding: 2rem;
	}

	.quiz-card-inner {
		position: relative;
		width: 100%;
		height: 100%;
		text-align: center;
		transition: transform 0.6s;
		transform-style: preserve-3d;
	}

	.quiz-card.flipped .quiz-card-inner {
		transform: rotateY(180deg);
	}

	.quiz-card-front,
	.quiz-card-back {
		position: absolute;
		width: 100%;
		height: 100%;
		-webkit-backface-visibility: hidden;
		backface-visibility: hidden;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		background: var(--bg-secondary);
		border: 1px solid var(--border-color);
		border-radius: 12px;
		padding: 2rem;
		color: var(--text-primary);
	}

	.quiz-card-back {
		transform: rotateY(180deg);
	}

	.quiz-hint {
		position: absolute;
		bottom: 1rem;
		left: 0;
		right: 0;
		text-align: center;
		color: var(--text-tertiary);
		font-size: 0.875rem;
	}

	.quiz-actions {
		position: absolute;
		bottom: 1.5rem;
		left: 0;
		right: 0;
		display: flex;
		justify-content: center;
		gap: 1rem;
		padding: 0 1.5rem;
	}

	.quiz-button {
		padding: 0.75rem 1.5rem;
		border: none;
		border-radius: 8px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		font-size: 0.875rem;
	}

	.quiz-button.correct {
		background-color: var(--success-color);
		color: white;
	}

	.quiz-button.incorrect {
		background-color: var(--danger-color);
		color: white;
	}

	.quiz-results {
		padding: 2rem;
	}

	.score-display {
		text-align: center;
		margin: 2rem 0;
	}

	.score {
		font-size: 3rem;
		font-weight: 700;
		color: var(--primary-color);
	}

	.percentage {
		font-size: 1.5rem;
		color: var(--text-secondary);
		margin-top: 0.5rem;
	}

	.results-breakdown {
		margin-top: 2rem;
	}

	.results-list {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		margin-top: 1rem;
		max-height: 300px;
		overflow-y: auto;
	}

	.result-item {
		display: flex;
		gap: 1rem;
		padding: 1rem;
		background: var(--bg-tertiary);
		border-radius: 8px;
		border: 1px solid var(--border-color);
	}

	.result-item.correct {
		border-left: 4px solid var(--success-color);
	}

	.result-item:not(.correct) {
		border-left: 4px solid var(--danger-color);
	}

	.result-content {
		flex: 1;
	}

	.result-question {
		font-weight: 500;
		margin-bottom: 0.5rem;
		color: var(--text-primary);
	}

	.result-answer {
		color: var(--text-secondary);
		font-size: 0.875rem;
	}

	.quiz-footer {
		margin-top: 2rem;
		display: flex;
		justify-content: flex-end;
		gap: 1rem;
	}

	.quiz-card.flipped .quiz-card-front p,
	.quiz-card:not(.flipped) .quiz-card-back p {
		opacity: 0;
		transition: opacity 0.1s;
	}

	.quiz-card-front p,
	.quiz-card-back p {
		opacity: 1;
		transition: opacity 0.3s;
	}

	.quiz-card.transitioning .quiz-card-front p,
	.quiz-card.transitioning .quiz-card-back p,
	.quiz-card.transitioning .quiz-hint,
	.quiz-card.transitioning .quiz-actions {
		opacity: 0 !important;
		transition: opacity 0.1s !important;
	}

	.quiz-card-front p,
	.quiz-card-back p,
	.quiz-hint,
	.quiz-actions {
		opacity: 1;
		transition: opacity 0.3s 0.3s; /* Delay showing content until flip completes */
	}

	/* Keep the Q/A labels visible */
	.quiz-card .card-label {
		opacity: 1 !important;
		transition: none !important;
	}

	.quiz-card.transitioning .quiz-card-inner *:not(.card-label) {
		opacity: 0 !important;
		transition: opacity 0s !important;
	}

	.quiz-card-front *:not(.card-label),
	.quiz-card-back *:not(.card-label) {
		opacity: 1;
		transition: opacity 0.3s 0.3s;
	}

	/* Keep the Q/A labels visible */
	.quiz-card .card-label {
		opacity: 1 !important;
		transition: none !important;
	}

	.lesson-controls .primary-button:first-child {
		background-color: #f59e0b;
		box-shadow: 0 2px 4px rgba(245, 158, 11, 0.2);
	}

	.lesson-controls .primary-button:first-child:hover {
		background-color: #d97706;
		box-shadow: 0 4px 8px rgba(245, 158, 11, 0.3);
	}

	.lesson-controls .primary-button:nth-child(2) {
		background-color: #f87171;  /* Light red for reverse quiz */
		box-shadow: 0 2px 4px rgba(248, 113, 113, 0.2);
	}

	.lesson-controls .primary-button:nth-child(2):hover {
		background-color: #ef4444;
		box-shadow: 0 4px 8px rgba(248, 113, 113, 0.3);
	}

	.quiz-cover {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: var(--bg-secondary);
		z-index: 100;
		border-radius: 12px;
	}

	.quiz-content {
		opacity: 1;
		transition: opacity 0.15s ease;  /* Quick fade transition */
	}

	.quiz-content.hidden {
		opacity: 0;
		transition: opacity 0.15s ease;  /* Same quick fade for hiding */
	}

	/* Remove all other transition styles for quiz card content */

	/* Add specific styling for quiz modal overlay */
	.quiz-modal-overlay {
		background-color: var(--bg-primary);
		background-image: 
			linear-gradient(45deg, var(--bg-secondary) 25%, transparent 25%),
			linear-gradient(-45deg, var(--bg-secondary) 25%, transparent 25%),
			linear-gradient(45deg, transparent 75%, var(--bg-secondary) 75%),
			linear-gradient(-45deg, transparent 75%, var(--bg-secondary) 75%);
		background-size: 20px 20px;
		background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
		opacity: 0.98;
	}

	.results-breakdown h3 {
		color: var(--text-primary);
		margin-bottom: 1rem;
	}

	.quiz-results h2,
	.quiz-results h3,
	.results-breakdown h3 {
		color: var(--text-primary);
		margin-bottom: 1rem;
	}

	.quiz-button {
		background-color: #f59e0b;
		box-shadow: 0 2px 4px rgba(245, 158, 11, 0.2);
	}

	.quiz-button:hover {
		background-color: #d97706;
		box-shadow: 0 4px 8px rgba(245, 158, 11, 0.3);
	}

	.reverse-quiz-button {
		background-color: #8b5cf6;  /* Purple for distinction */
		box-shadow: 0 2px 4px rgba(139, 92, 246, 0.2);
	}

	.reverse-quiz-button:hover {
		background-color: #7c3aed;
		box-shadow: 0 4px 8px rgba(139, 92, 246, 0.3);
	}

	.lesson-controls {
		display: flex;
		align-items: center;
		gap: 0.75rem;
	}

	.lesson-controls .primary-button {
		background-color: var(--primary-color);
		box-shadow: 0 2px 4px rgba(59, 130, 246, 0.2);
	}

	.lesson-controls .primary-button:hover {
		background-color: var(--primary-hover);
		box-shadow: 0 4px 8px rgba(59, 130, 246, 0.3);
	}

	/* Remove the quiz-specific button styles */
	.quiz-button,
	.reverse-quiz-button {
		/* Remove these styles */
	}

	.lesson-title-row {
		display: flex;
		justify-content: flex-end;  /* Align buttons to the right */
		padding: 0.75rem 0.5rem;
		margin-bottom: 2rem;
		border-bottom: 2px solid var(--border-color);
		position: relative;
	}

	/* ... rest of the styles ... */
</style>
