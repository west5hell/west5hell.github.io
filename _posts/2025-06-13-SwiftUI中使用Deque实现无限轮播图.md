---
layout: post
title: "SwiftUI 中使用 Deque 实现无限轮播图"
date: 2025-06-13 18:20:00 +0800
categories: SwiftUI
---

# SwiftUI 中使用 Deque 实现无限轮播图

一个简单的轮播图实现，数据结构使用 Dqeue 而不是传统的 Array

> 需要使用到苹果的集合库 [Swift Collections](https://github.com/apple/swift-collections)

```swift
import Collections
import SwiftUI

struct InfiniteCarouselDequeView: View {
    private let originalItems: [String] = ["interzoo_2016", "interzoo_2024", "interzoo_2017", "interzoo_2024", "interzoo_2017"]
    @State private var deque: Deque<Int>
    @State private var xOffset: CGFloat = 0
    @GestureState private var dragOffset: CGFloat = 0
    @State private var timer: Timer?
    @State private var isDragging = false

    private let timerInterval: TimeInterval = 3
    private var currentPageIndex: Int {
        return deque.first ?? 0
    }

    init() {
        self._deque = State(initialValue: Deque(0..<originalItems.count))
    }

    var body: some View {
        GeometryReader { geometry in
            let width = geometry.size.width

            HStack(spacing: 0) {
                ForEach(visibleItems(), id: \.self) { item in
                    Image(item)
                        .resizable()
                        .scaledToFill()
                        .frame(width: width, height: geometry.size.height)
                        .clipped()
                }
            }
            .offset(x: xOffset + dragOffset - width)
            .gesture(
                DragGesture()
                    .onChanged { _ in
                        pauseTimer()
                        isDragging = true
                    }
                    .updating($dragOffset) { value, state, _ in
                        state = value.translation.width
                    }
                    .onEnded { value in
                        let threshold = width / 3
                        let translation = value.translation.width
                        isDragging = false

                        if translation < -threshold {
                            animateToNext(width: width)
                        } else if translation > threshold {
                            animateToPrevious(width: width)
                        } else {
                            withAnimation {
                                xOffset = 0
                            }
                        }

                        resumeTimer(width: width)
                    }
            )
            .onAppear {
                startTimer(width: width)
            }
            .onDisappear {
                stopTimer()
            }
        }
        .frame(height: 200)
        .overlay(
            HStack(spacing: 8) {
                ForEach(originalItems.indices, id: \.self) { i in
                    Circle()
                        .fill(i == currentPageIndex ? .blue : .gray)
                        .frame(width: 8, height: 8)
                }
            }
            .padding(.bottom, 10),
            alignment: .bottom
        )
    }

    // MARK: - Get visible items (3 items centered around deque[0])
    private func visibleItems() -> [String] {
        let items = Array(deque)
        if items.count >= 3 {
            return [
                originalItems[items[(items.count - 1) % items.count]],
                originalItems[items[0]],
                originalItems[items[1 % items.count]]
            ]
        } else if items.count == 2 {
            return [
                originalItems[items[1]],
                originalItems[items[0]],
                originalItems[items[1]]
            ]
        } else if items.count == 1 {
            return Array(repeating: originalItems[items[0]], count: 3)
        } else {
            return Array(repeating: "", count: 3)
        }
    }

    // MARK: - Carousel Transitions

    private func animateToNext(width: CGFloat) {
        withAnimation(.easeInOut(duration: 0.3)) {
            xOffset = -width
        }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
            // Rotate left
            let first = deque.removeFirst()
            deque.append(first)
            xOffset = 0
        }
    }

    private func animateToPrevious(width: CGFloat) {
        withAnimation(.easeInOut(duration: 0.3)) {
            xOffset = width
        }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
            // Rotate right
            let last = deque.removeLast()
            deque.prepend(last)
            xOffset = 0
        }
    }

    // MARK: - Timer Management

    private func startTimer(width: CGFloat) {
        timer = Timer.scheduledTimer(
            withTimeInterval: timerInterval,
            repeats: true
        ) { _ in
            guard !isDragging else { return }
            animateToNext(width: width)
        }
    }

    private func pauseTimer() {
        timer?.invalidate()
    }

    private func resumeTimer(width: CGFloat) {
        timer?.invalidate()
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
            startTimer(width: width)
        }
    }

    private func stopTimer() {
        timer?.invalidate()
        timer = nil
    }
}
```

### 注意

> 手动滑动的时候还有一点小瑕疵，但是影响不大，可以自行修改。

## 结语

在 `UIKit` 中应该也可以使用 `Deque` 来实现同样的效果
