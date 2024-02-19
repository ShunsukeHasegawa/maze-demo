<template>
  <main>
    <div id="options">
      <div class="option-field">
        <label>生成方法</label>
        <select v-model="makeMethod">
          <option value="extend">壁伸ばし法</option>
          <option value="bar">棒倒し法</option>
          <option value="dig">穴掘り法</option>
        </select>
        <button @click="makeMaze" v-bind:disabled="isSearching">
          {{ isMaking ? '停止' : '生成' }}
        </button>
      </div>
      <div class="option-field">
        <label>探索方法</label>
        <select v-model="findMethod">
          <option value="depth" selected>深さ優先探索</option>
          <option value="width">幅優先探索</option>
        </select>
        <button @click="find" v-bind:disabled="isMaking && !isSearching">
          {{ isSearching ? '停止' : '探索' }}
        </button>
      </div>
      <div class="option-field">
        <label>描画速度</label>
        <input type="range" v-model="wait" min="0" max="100" step="10" />
      </div>
    </div>
    <div id="maize">
      <table>
        <tr v-for="(row, y) in mazeArray" :key="y">
          <td
            v-for="(cell, x) in row"
            :key="x"
            :class="getCellClass(x, y)"
            @click="setStart"
            @contextmenu.prevent="setGoal"
          >
            <span v-if="start.x === x && start.y === y" class="start">S</span>
            <span v-else-if="goal.x === x && goal.y === y" class="goal">G</span>
          </td>
        </tr>
      </table>
      <p>通路の部分を左クリックでスタート地点を、右クリックでゴール地点を設定できます。</p>
    </div>
  </main>
</template>

<script>
export default {
  name: 'App',
  components: {},
  data() {
    return {
      mazeArray: [],
      width: 31,
      height: 31,
      start: { x: 1, y: 1 },
      goal: { x: 1, y: 1 },
      wait: 0,
      isMaking: false,
      isSearching: false,
      size: 'm',
      makeMethod: 'extend',
      findMethod: 'depth',
      current: { x: 1, y: 1 }
    }
  },
  mounted() {
    this.makeMaze()
    this.wait = 10
  },
  methods: {
    getCellClass(x, y) {
      const val = this.mazeArray[y][x]
      let className = ''
      switch (val) {
        case 0:
          className = 'path'
          break
        case 1:
          className = 'wall'
          break
        case 2:
          className = 'visited'
          break
        case 3:
          return 'current'
      }

      if (this.current.x === x && this.current.y === y) {
        className += ' current'
      }

      return className
    },
    async makeMaze() {
      if (this.isMaking) {
        this.isMaking = false
        return
      }

      this.isMaking = true
      switch (this.makeMethod) {
        case 'extend':
          await this.mazeExtend()
          break
        case 'bar':
          await this.mazeBar()
          break
        case 'dig':
          await this.mazeDig()
          break
      }
      this.isMaking = false
    },
    async mazeExtend() {
      this.initMazeArray()
      // 壁伸ばし法に基づいて迷路を生成する
      for (let y = 2; y < this.height - 2; y += 2) {
        for (let x = 2; x < this.width - 2; x += 2) {
          if (this.wait > 0) {
            await new Promise((resolve) => setTimeout(resolve, this.wait))
          }

          if (this.isMaking === false) return

          this.mazeArray[y][x] = 1
          let direction = Math.floor(Math.random() * 4)
          switch (direction) {
            case 0: // 上
              this.mazeArray[y - 1][x] = 1
              break
            case 1: // 右
              this.mazeArray[y][x + 1] = 1
              break
            case 2: // 下
              this.mazeArray[y + 1][x] = 1
              break
            case 3: // 左
              this.mazeArray[y][x - 1] = 1
              break
          }
        }
      }
    },
    async mazeBar() {
      this.initMazeArray()
      // 棒倒し法に基づいて迷路を生成する
      for (let y = 0; y < this.height; y++) {
        for (let x = 0; x < this.width; x++) {
          if (y === 0 || x === 0 || x === this.width - 1 || y === this.height - 1) {
            this.mazeArray[y][x] = 1
          } else if (y % 2 === 0 && x % 2 === 0) {
            this.mazeArray[y][x] = 1
          }
        }
      }

      for (let y = 2; y < this.height - 2; y += 2) {
        for (let x = 2; x < this.width - 2; x += 2) {
          if (this.wait > 0) {
            await new Promise((resolve) => setTimeout(resolve, this.wait))
          }

          if (this.isMaking === false) return

          this.mazeArray[y][x] = 1
          let direction = Math.floor(Math.random() * 2)
          switch (direction) {
            case 0: // 横
              this.mazeArray[y][x + 1] = 1
              break
            case 1: // 縦
              this.mazeArray[y + 1][x] = 1
              break
          }
        }
      }
    },
    async mazeDig() {
      this.initMazeArray() // 迷路の初期化
      // 全てのセルを壁として初期化
      for (let y = 0; y < this.height; y++) {
        for (let x = 0; x < this.width; x++) {
          this.mazeArray[y][x] = 1
        }
      }

      let stack = [] // 掘り進めるセルのスタック
      stack.push({ x: 1, y: 1 }) // スタート地点をスタックに追加
      while (stack.length > 0) {
        // スタックが空になるまで繰り返す
        if (this.isMaking === false) return

        if (this.wait > 0) {
          await new Promise((resolve) => setTimeout(resolve, this.wait))
        }

        let current = stack[stack.length - 1] // 現在のセルを取得
        this.mazeArray[current.y][current.x] = 0 // 現在のセルを掘る

        // 上下左右の2セル先が壁であるセルを取得
        let next = this.getDigNext(current.x, current.y)

        if (next) {
          stack.push(next) // 次のセルがあればスタックに追加
        } else {
          stack.pop() // 次のセルがなければ一つ前のセルに戻る
        }
      }
    },
    getDigNext(x, y) {
      let directions = [
        { x: 0, y: -2 },
        { x: 2, y: 0 },
        { x: 0, y: 2 },
        { x: -2, y: 0 }
      ]
      let nexts = []
      for (let direction of directions) {
        let nextX = x + direction.x
        let nextY = y + direction.y
        if (
          nextX > 0 &&
          nextX < this.width - 1 &&
          nextY > 0 &&
          nextY < this.height - 1 &&
          this.mazeArray[nextY][nextX] === 1
        ) {
          nexts.push({ x: nextX, y: nextY })
        }
      }
      if (nexts.length > 0) {
        let next = nexts[Math.floor(Math.random() * nexts.length)]
        let wallX = (x + next.x) / 2
        let wallY = (y + next.y) / 2
        this.mazeArray[wallY][wallX] = 0
        return next
      }
      return null
    },
    initMazeArray() {
      this.start = { x: 1, y: 1 }
      this.goal = { x: this.width - 2, y: this.height - 2 }
      this.mazeArray = []
      for (let y = 0; y < this.height; y++) {
        this.mazeArray[y] = []
        for (let x = 0; x < this.width; x++) {
          // 外周は必ず壁とする
          if (y === 0 || x === 0 || x === this.width - 1 || y === this.height - 1) {
            this.mazeArray[y][x] = 1
          } else {
            this.mazeArray[y][x] = 0
          }
        }
      }
    },
    setStart(e) {
      // もし壁なら何もしない
      if (e.target.classList.contains('wall')) return
      this.start = { x: e.target.cellIndex, y: e.target.parentNode.rowIndex }
    },
    setGoal(e) {
      if (e.target.classList.contains('wall')) return
      this.goal = { x: e.target.cellIndex, y: e.target.parentNode.rowIndex }
    },
    find() {
      if (this.isSearching) {
        this.stopFind()
        return
      }
      switch (this.findMethod) {
        case 'depth':
          this.findByDepth()
          break
        case 'width':
          this.findByWidth()
          break
      }
    },
    async findByDepth() {
      this.stopFind()
      this.isSearching = true

      // スタート地点からゴール地点までを「深さ優先探索」で探索する
      // 1. 現在位置を既に通ったマスに設定した後、現在位置から進めるマス（壁ではない and 既に通ったマスではない）を探す。
      // 2. 進めるマスがあれば、現在位置をスタックに格納し、現在位置を進めるマスの位置に変更する。なお、進めるマスが見つかった時点で、進めるマスの候補の探索は終了する。
      // 3. 進めるマスがなければ、スタックから進む前の位置を取り出し、現在位置を元の位置へ戻す（＝引き返す処理）。
      // 4. ゴールするまで、1～3を繰り返す。

      let stack = []
      stack.push(this.start)
      while (stack.length > 0) {
        if (this.isSearching === false) return

        let current = stack[stack.length - 1]
        this.current = current

        // 現在地を通ったマスに設定
        this.mazeArray[current.y][current.x] = 2

        if (this.wait > 0) {
          await new Promise((resolve) => setTimeout(resolve, this.wait))
        }

        if (current.x === this.goal.x && current.y === this.goal.y) {
          this.markRouteDepth(stack)
          this.isSearching = false
          alert('ゴール!')
          return
        }

        // 上下左右のマスを探索

        // 右
        if (this.nextValidate(current.x + 1, current.y)) {
          stack.push({ x: current.x + 1, y: current.y })
          continue
        }

        // 下
        if (this.nextValidate(current.x, current.y + 1)) {
          stack.push({ x: current.x, y: current.y + 1 })
          continue
        }

        // 左
        if (this.nextValidate(current.x - 1, current.y)) {
          stack.push({ x: current.x - 1, y: current.y })
          continue
        }

        // 上
        if (this.nextValidate(current.x, current.y - 1)) {
          stack.push({ x: current.x, y: current.y - 1 })
          continue
        }

        // 上下左右のマスが全て通れない場合、スタックから取り出す
        stack.pop()
      }

      // ゴール地点にたどり着けなかった場合
      alert('No path found')
      this.isSearching = false
    },
    async findByWidth() {
      this.stopFind()
      this.isSearching = true

      // スタート地点からゴール地点までを「幅優先探索」で探索する
      // 1. 現在位置を既に通ったマスに設定した後、現在位置から進めるマス（壁ではない and 既に通ったマスではない）を探す。
      // 2. 進めるマスがあれば、それら全てをキューに格納する。
      //    さらに、進んだマスがどのマスから来たのか分かるように、現在位置の情報を記録しておく（最短経路の導出に利用）。
      // 3. キューから進めるマスの位置を取り出し、現在位置とする。
      // 4. ゴールするまで、1～3を繰り返す。

      let queue = []
      let parent = {}
      queue.push(this.start)
      parent[`${this.start.x},${this.start.y}`] = null

      while (queue.length > 0) {
        if (this.isSearching === false) return

        let current = queue.shift()
        this.current = current

        // 現在地を通ったマスに設定
        this.mazeArray[current.y][current.x] = 2

        if (this.wait > 0) {
          await new Promise((resolve) => setTimeout(resolve, this.wait))
        }

        if (current.x === this.goal.x && current.y === this.goal.y) {
          this.markRouteWidth(parent, current)
          this.isSearching = false
          alert('ゴール!')
          return
        }

        // 上下左右のマスを探索
        // 右
        if (this.nextValidate(current.x + 1, current.y)) {
          queue.push({ x: current.x + 1, y: current.y })
          this.mazeArray[current.y][current.x + 1] = 2
          parent[`${current.x + 1},${current.y}`] = current
        }

        // 下
        if (this.nextValidate(current.x, current.y + 1)) {
          queue.push({ x: current.x, y: current.y + 1 })
          this.mazeArray[current.y + 1][current.x] = 2
          parent[`${current.x},${current.y + 1}`] = current
        }

        // 左
        if (this.nextValidate(current.x - 1, current.y)) {
          queue.push({ x: current.x - 1, y: current.y })
          this.mazeArray[current.y][current.x - 1] = 2
          parent[`${current.x - 1},${current.y}`] = current
        }

        // 上
        if (this.nextValidate(current.x, current.y - 1)) {
          queue.push({ x: current.x, y: current.y - 1 })
          this.mazeArray[current.y - 1][current.x] = 2
          parent[`${current.x},${current.y - 1}`] = current
        }
      }

      // ゴール地点にたどり着けなかった場合
      alert('No path found')
      this.isSearching = false
    },
    markRouteDepth(stack) {
      stack.forEach((cell, index) => {
        if (index === 0) return
        this.mazeArray[cell.y][cell.x] = 3
      })
    },
    markRouteWidth(parent, current) {
      let path = []
      while (current != null) {
        path.unshift(current)
        current = parent[`${current.x},${current.y}`]
        if (current != null) {
          this.mazeArray[current.y][current.x] = 3
        }
      }
    },
    nextValidate(x, y) {
      return this.mazeArray[y][x] === 0
    },
    stopFind() {
      this.isSearching = false
      for (let y = 0; y < this.height; y++) {
        for (let x = 0; x < this.width; x++) {
          if (this.mazeArray[y][x] > 1) {
            this.mazeArray[y][x] = 0
          }
        }
      }
    }
  },
  watch: {
    size() {
      switch (this.size) {
        case 's':
          this.width = 11
          this.height = 11
          break
        case 'm':
          this.width = 31
          this.height = 31
          break
        case 'l':
          this.width = 51
          this.height = 51
          break
      }
    }
  }
}
</script>
