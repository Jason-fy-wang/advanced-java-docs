---
tags:
  - frontend
  - React
  - unittest
---
针对环境为 vite + react  的unit test, 总体流程和 vite+vue很相似.
[[6-how-to-do-frontend-unittest]]


#### step1: install dependency
```shell
# vitest
npm install -D vitest jsdom

# React Testing Library
npm install -D @testing-library/react @testing-library/jest-dom

# ts
npm install -D @types/testing-library__jest-dom

```


#### step2: vitest config
```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,            // 开启全局模式（可直接用 describe、it）
    environment: 'jsdom',     // 浏览器模拟环境
    setupFiles: './vitest.setup.ts',
    css: false,               // 如不需要处理 CSS
  }
})


# vitest.setup.ts
import '@testing-library/jest-dom'

```


#### step3:  unit test
```js
import { useState } from 'react'

export default function Counter() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <span data-testid="count">{count}</span>
    </div>
  )
}

```


unit test:
```js
import { render, screen, fireEvent } from '@testing-library/react'
import Counter from './Counter'

describe('Counter 组件', () => {
  it('初始值为 0', () => {
    render(<Counter />)
    expect(screen.getByTestId('count')).toHaveTextContent('0')
  })

  it('点击按钮后数字递增', async () => {
    render(<Counter />)
    const btn = screen.getByRole('button', { name: '+' })
    await fireEvent.click(btn)
    expect(screen.getByTestId('count')).toHaveTextContent('1')
  })
})

```

#### step4: unit test file
```js
{
  "scripts": {
    "test": "vitest run --reporter=verbose",
    "test:ui": "vitest --ui",   // 可视化测试界面
    "test:coverage": "vitest run --coverage"
  }
}

```




