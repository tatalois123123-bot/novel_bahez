import React, { useState, useCallback, useMemo, useEffect } from 'react';
import Header from './components/Header';
import Sidebar from './components/Sidebar';
import ReaderView from './components/ReaderView';
import GeneralReaderView from './components/GeneralReaderView';
import ChapterEditor from './components/ChapterEditor';
import SearchPanel from './components/SearchPanel';
import StatusPanel from './components/StatusPanel';
import BookmarkPanel from './components/BookmarkPanel';
import LeftToolbar from './components/LeftToolbar';
import { MOCK_NOVEL } from './constants';
import { useSettings } from './hooks/useSettings';
import type { Chapter } from './types';

// Helper function to count words from HTML content
const countWords = (htmlString: string) => {
    const text = htmlString.replace(/<[^>]+>/g, ' ');
    return text.trim().split(/\s+/).filter(Boolean).length;
};

const App: React.FC = () => {
  const [novel, setNovel] = useState(() => {
    try {
      const savedNovel = localStorage.getItem('kurdish-novel-data');
      return savedNovel ? JSON.parse(savedNovel) : MOCK_NOVEL;
    } catch (error) {
      console.error("Failed to parse novel from localStorage", error);
      return MOCK_NOVEL;
    }
  });

  const [currentChapterIndex, setCurrentChapterIndex] = useState(() => {
    try {
        const savedNovel = localStorage.getItem('kurdish-novel-data');
        const novelData = savedNovel ? JSON.parse(savedNovel) : MOCK_NOVEL;
        
        const savedIndex = localStorage.getItem('novel-reader-last-chapter');
        if (savedIndex) {
            const index = parseInt(savedIndex, 10);
            if (index >= 0 && index < novelData.chapters.length) {
                return index;
            }
        }
    } catch (error) {
        console.error("Failed to read last chapter index from localStorage", error);
    }
    return 0;
  });
  
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const [isEditorOpen, setIsEditorOpen] = useState(false);
  const [isSearchPanelOpen, setIsSearchPanelOpen] = useState(false);
  const [isStatusPanelOpen, setIsStatusPanelOpen] = useState(false);
  const [isBookmarkPanelOpen, setIsBookmarkPanelOpen] = useState(false);
  const [chapterToEdit, setChapterToEdit] = useState<Chapter | null>(null);
  const [scrollPercentage, setScrollPercentage] = useState(0);
  const { settings, updateSettings, getReaderViewProseClass, getGeneralViewProseClass } = useSettings();

  const [highlightInfo, setHighlightInfo] = useState<{
    chapterId: number;
    resultIndex: number;
    searchTerm: string;
  } | null>(null);

  const [isGeneralViewActive, setIsGeneralViewActive] = useState(false);
  
  // Persist novel data to localStorage whenever it changes
  useEffect(() => {
    try {
      localStorage.setItem('kurdish-novel-data', JSON.stringify(novel));
    } catch (error) {
      console.error("Failed to save novel to localStorage", error);
    }
  }, [novel]);

  // Save last read chapter index
  useEffect(() => {
    localStorage.setItem('novel-reader-last-chapter', String(currentChapterIndex));
  }, [currentChapterIndex]);


  const currentChapter = novel.chapters[currentChapterIndex];

  const readingStats = useMemo(() => {
    const chapterWordCounts = novel.chapters.map(c => countWords(c.content));
    const totalNovelWords = chapterWordCounts.reduce((sum, count) => sum + count, 0);

    const wordsInPreviousChapters = chapterWordCounts
        .slice(0, currentChapterIndex)
        .reduce((sum, count) => sum + count, 0);
    
    const wordsInCurrentChapter = chapterWordCounts[currentChapterIndex] || 0;
    const wordsScrolledInCurrent = Math.round((scrollPercentage / 100) * wordsInCurrentChapter);

    const totalWordsRead = wordsInPreviousChapters + wordsScrolledInCurrent;

    const overallProgress = totalNovelWords > 0 ? (totalWordsRead / totalNovelWords) * 100 : 0;
    
    const bookmarkedChaptersCount = novel.chapters.filter(c => c.bookmarked).length;

    return {
        overallProgress: Math.round(overallProgress),
        chapterProgress: Math.round(scrollPercentage),
        totalWordsRead,
        totalNovelWords,
        totalChapters: novel.chapters.length,
        bookmarkedChapters: bookmarkedChaptersCount,
    };
  }, [novel.chapters, currentChapterIndex, scrollPercentage]);

  const bookmarkedChapters = useMemo(() => {
    return novel.chapters.filter(c => c.bookmarked);
  }, [novel.chapters]);

  const handleSelectChapter = useCallback((chapterId: number) => {
    const chapterIndex = novel.chapters.findIndex(c => c.id === chapterId);
    if (chapterIndex !== -1) {
        setCurrentChapterIndex(chapterIndex);
    }
    setHighlightInfo(null); // Clear highlight when manually selecting a chapter
    setIsSidebarOpen(false);
    setIsBookmarkPanelOpen(false);
    setIsGeneralViewActive(false); // Exit general view when a chapter is selected
  }, [novel.chapters]);

  const handleSelectSearchResult = useCallback((chapterId: number, resultIndex: number, searchTerm: string) => {
      const chapterIndex = novel.chapters.findIndex(c => c.id === chapterId);
      if (chapterIndex !== -1) {
          setCurrentChapterIndex(chapterIndex);
          setHighlightInfo({ chapterId, resultIndex, searchTerm });
          setIsGeneralViewActive(false);
          setIsSearchPanelOpen(false);
      }
  }, [novel.chapters]);
  
  const handleToggleGeneralView = () => {
    setIsGeneralViewActive(prev => {
        const nextState = !prev;
        if (nextState) {
            // When entering general view, close other panels
            setIsSidebarOpen(false);
            setIsStatusPanelOpen(false);
            setIsBookmarkPanelOpen(false);
        }
        return nextState;
    });
  };

  const handleNextChapter = useCallback(() => {
    setCurrentChapterIndex(prev => Math.min(prev + 1, novel.chapters.length - 1));
  }, [novel.chapters.length]);

  const handlePreviousChapter = useCallback(() => {
    setCurrentChapterIndex(prev => Math.max(prev - 1, 0));
  }, []);

  const handleToggleBookmark = useCallback((chapterId: number) => {
    setNovel(prevNovel => {
      const updatedChapters = prevNovel.chapters.map(chapter =>
        chapter.id === chapterId
          ? { ...chapter, bookmarked: !chapter.bookmarked }
          : chapter
      );
      return { ...prevNovel, chapters: updatedChapters };
    });
  }, []);
  
  const handleToggleCurrentChapterBookmark = useCallback(() => {
    if (currentChapter) {
        handleToggleBookmark(currentChapter.id);
    }
  }, [handleToggleBookmark, currentChapter]);

  const handleUpdateNote = useCallback((chapterId: number, note: string) => {
    setNovel(prevNovel => {
        const updatedChapters = prevNovel.chapters.map(chapter => 
            chapter.id === chapterId ? { ...chapter, notes: note } : chapter
        );
        return { ...prevNovel, chapters: updatedChapters };
    });
  }, []);

  const handleOpenEditorForNew = useCallback(() => {
    setChapterToEdit(null);
    setIsEditorOpen(true);
  }, []);

  const handleOpenEditorForEdit = useCallback((chapterId: number) => {
    const chapter = novel.chapters.find(c => c.id === chapterId);
    if (chapter) {
      setChapterToEdit(chapter);
      setIsEditorOpen(true);
    }
  }, [novel.chapters]);

  const handleCloseEditor = useCallback(() => {
    setIsEditorOpen(false);
    setChapterToEdit(null);
  }, []);

  const handleSaveChapter = useCallback((chapterData: { title: string; content: string; id?: number }) => {
    setNovel(prevNovel => {
      if (chapterData.id) {
        // Update existing chapter
        const updatedChapters = prevNovel.chapters.map(chapter =>
          chapter.id === chapterData.id
            ? { ...chapter, title: chapterData.title, content: chapterData.content }
            : chapter
        );
        return { ...prevNovel, chapters: updatedChapters };
      } else {
        // Add new chapter
        const newChapter: Chapter = {
          id: (prevNovel.chapters[prevNovel.chapters.length - 1]?.id || 0) + 1,
          title: chapterData.title,
          content: chapterData.content,
          bookmarked: false,
          notes: '',
        };
        const updatedChapters = [...prevNovel.chapters, newChapter];
        return { ...prevNovel, chapters: updatedChapters };
      }
    });
    handleCloseEditor();
  }, [handleCloseEditor]);
  
  const handleDeleteChapter = useCallback((chapterId: number) => {
    if (window.confirm('ئایا دڵنیایت لە سڕینەوەی ئەم بەشە؟ ئەم کارە ناگەڕێتەوە.')) {
        setNovel(prevNovel => {
            const chapterToDeleteIndex = prevNovel.chapters.findIndex(c => c.id === chapterId);
            if (chapterToDeleteIndex === -1) return prevNovel;

            const updatedChapters = prevNovel.chapters.filter(c => c.id !== chapterId);
            
            if (updatedChapters.length === 0) {
                setCurrentChapterIndex(0); 
                return { ...prevNovel, chapters: [] };
            }

            if (currentChapterIndex === chapterToDeleteIndex) {
                setCurrentChapterIndex(Math.max(0, chapterToDeleteIndex - 1));
            } else if (currentChapterIndex > chapterToDeleteIndex) {
                setCurrentChapterIndex(prev => prev - 1);
            }

            return { ...prevNovel, chapters: updatedChapters };
        });
    }
}, [currentChapterIndex]);

  if (!currentChapter && !isGeneralViewActive && novel.chapters.length === 0) {
    return (
        <div className="flex items-center justify-center h-screen bg-bg-primary text-text-primary">
            <div className="text-center">
                <h1 className="text-2xl font-bold mb-4">هیچ بەشێک بوونی نییە</h1>
                <p className="text-text-secondary mb-6">بۆ دەستپێکردن، بەشێکی نوێ زیاد بکە.</p>
                <button
                    onClick={handleOpenEditorForNew}
                    className="px-6 py-2 rounded-md bg-accent text-white hover:opacity-90 transition-opacity"
                >
                    بەشی نوێ زیاد بکە
                </button>
                {isEditorOpen && (
                    <ChapterEditor
                        onSave={handleSaveChapter}
                        onClose={handleCloseEditor}
                        chapterToEdit={chapterToEdit}
                    />
                )}
            </div>
        </div>
    );
  }

  return (
    <div className="flex h-screen font-sans">
      <LeftToolbar 
        settings={settings} 
        onUpdateSettings={updateSettings}
        isGeneralViewActive={isGeneralViewActive}
        onToggleGeneralView={handleToggleGeneralView} 
      />
      <div className="flex flex-1 lg:pl-24">
          <Sidebar
            chapters={novel.chapters}
            currentChapterId={currentChapter?.id}
            onSelectChapter={handleSelectChapter}
            isOpen={isSidebarOpen}
            onClose={() => setIsSidebarOpen(false)}
            novelTitle={novel.title}
            onAddNewChapter={handleOpenEditorForNew}
            onEditChapter={handleOpenEditorForEdit}
            onToggleBookmark={handleToggleBookmark}
            onDeleteChapter={handleDeleteChapter}
          />
        <div className="flex flex-col flex-1 w-full lg:w-0 h-screen">
          <Header
            novelTitle={novel.title}
            chapterTitle={(isGeneralViewActive || !currentChapter) ? "خوێندنەوەی گشتی" : currentChapter.title}
            onToggleSidebar={() => setIsSidebarOpen(prev => !prev)}
            onToggleSearch={() => setIsSearchPanelOpen(prev => !prev)}
            onToggleStatusPanel={() => setIsStatusPanelOpen(prev => !prev)}
            onToggleBookmarkPanel={() => setIsBookmarkPanelOpen(prev => !prev)}
            settings={settings}
            onUpdateSettings={updateSettings}
            hasBookmarks={bookmarkedChapters.length > 0}
            isGeneralViewActive={isGeneralViewActive}
          />
          {isGeneralViewActive || !currentChapter ? (
            <GeneralReaderView chapters={novel.chapters} proseClass={getGeneralViewProseClass()} onSelectChapter={handleSelectChapter} />
          ) : (
            <ReaderView
              chapter={currentChapter}
              proseClass={getReaderViewProseClass()}
              settings={settings}
              onUpdateSettings={updateSettings}
              onNext={handleNextChapter}
              onPrevious={handlePreviousChapter}
              onUpdateNote={handleUpdateNote}
              onScrollProgress={setScrollPercentage}
              isFirstChapter={currentChapterIndex === 0}
              isLastChapter={currentChapterIndex === novel.chapters.length - 1}
              highlightInfo={highlightInfo}
              onHighlightComplete={() => setHighlightInfo(null)}
            />
          )}
        </div>
      </div>
      {isEditorOpen && (
        <ChapterEditor
          onSave={handleSaveChapter}
          onClose={handleCloseEditor}
          chapterToEdit={chapterToEdit}
        />
      )}
      <StatusPanel 
        isOpen={isStatusPanelOpen}
        onClose={() => setIsStatusPanelOpen(false)}
        stats={readingStats}
      />
      <SearchPanel 
        isOpen={isSearchPanelOpen}
        onClose={() => setIsSearchPanelOpen(false)}
        chapters={novel.chapters}
        onSelectResult={handleSelectSearchResult}
      />
      {currentChapter && (
        <BookmarkPanel
            isOpen={isBookmarkPanelOpen}
            onClose={() => setIsBookmarkPanelOpen(false)}
            bookmarkedChapters={bookmarkedChapters}
            onSelectChapter={handleSelectChapter}
            isCurrentChapterBookmarked={currentChapter.bookmarked}
            onToggleCurrentChapterBookmark={handleToggleCurrentChapterBookmark}
            currentChapterTitle={currentChapter.title}
        />
      )}
    </div>
  );
};

export default App;
