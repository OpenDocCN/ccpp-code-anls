# `whisper.cpp\examples\wchess\wchess.wasm\chessboardjs-1.0.0\js\chessboard-1.0.0.js`

```cpp
// chessboard.js v1.0.0
// 版本信息和来源链接
// https://github.com/oakmac/chessboardjs/
//
// 版权声明
// Copyright (c) 2019, Chris Oakman
// 使用 MIT 许可证发布
// https://github.com/oakmac/chessboardjs/blob/master/LICENSE.md

// 开始匿名作用域
;(function () {
  'use strict'

  // 获取全局变量 jQuery
  var $ = window['jQuery']

  // ---------------------------------------------------------------------------
  // 常量
  // ---------------------------------------------------------------------------

  // 棋盘列标识
  var COLUMNS = 'abcdefgh'.split('')
  // 默认拖动节流速率
  var DEFAULT_DRAG_THROTTLE_RATE = 20
  // 省略符号
  var ELLIPSIS = '…'
  // 最小的 jQuery 版本要求
  var MINIMUM_JQUERY_VERSION = '1.8.3'
  // 是否运行断言
  var RUN_ASSERTS = false
  // 初始 FEN
  var START_FEN = 'rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR'
  // 初始棋盘位置
  var START_POSITION = fenToObj(START_FEN)

  // 默认动画速度
  var DEFAULT_APPEAR_SPEED = 200
  var DEFAULT_MOVE_SPEED = 200
  var DEFAULT_SNAPBACK_SPEED = 60
  var DEFAULT_SNAP_SPEED = 30
  var DEFAULT_TRASH_SPEED = 100

  // 使用独特的类名以防止与页面上的其他内容冲突
  // 简化选择器
  // 注意：这些类名不应更改
  var CSS = {}
  CSS['alpha'] = 'alpha-d2270'
  CSS['black'] = 'black-3c85d'
  CSS['board'] = 'board-b72b1'
  CSS['chessboard'] = 'chessboard-63f37'
  CSS['clearfix'] = 'clearfix-7da63'
  CSS['highlight1'] = 'highlight1-32417'
  CSS['highlight2'] = 'highlight2-9c5d2'
  CSS['notation'] = 'notation-322f9'
  CSS['numeric'] = 'numeric-fc462'
  CSS['piece'] = 'piece-417db'
  CSS['row'] = 'row-5277c'
  CSS['sparePieces'] = 'spare-pieces-7492f'
  CSS['sparePiecesBottom'] = 'spare-pieces-bottom-ae20f'
  CSS['sparePiecesTop'] = 'spare-pieces-top-4028b'
  CSS['square'] = 'square-55d63'
  CSS['white'] = 'white-1e1d7'

  // ---------------------------------------------------------------------------
  // 杂项工具函数
  // ---------------------------------------------------------------------------

  // 节流函数
  function throttle (f, interval, scope) {
    var timeout = 0
    var shouldFire = false
    var args = []
    // 定义一个函数，处理超时事件
    var handleTimeout = function () {
      // 重置超时标记
      timeout = 0
      // 如果需要触发事件，则执行事件处理函数
      if (shouldFire) {
        shouldFire = false
        fire()
      }
    }

    // 定义一个函数，触发事件
    var fire = function () {
      // 设置新的超时事件
      timeout = window.setTimeout(handleTimeout, interval)
      // 调用传入的函数，并传入指定的作用域和参数
      f.apply(scope, args)
    }

    // 返回一个函数，用于执行节流操作
    return function (_args) {
      // 保存传入的参数
      args = arguments
      // 如果没有超时事件，则立即触发事件
      if (!timeout) {
        fire()
      } else {
        // 如果已经有超时事件，则标记需要触发事件
        shouldFire = true
      }
    }
  }

  // function debounce (f, interval, scope) {
  //   var timeout = 0
  //   return function (_args) {
  //     window.clearTimeout(timeout)
  //     var args = arguments
  //     timeout = window.setTimeout(function () {
  //       f.apply(scope, args)
  //     }, interval)
  //   }
  // }

  // 生成一个随机的 UUID
  function uuid () {
    return 'xxxx-xxxx-xxxx-xxxx-xxxx-xxxx-xxxx-xxxx'.replace(/x/g, function (c) {
      var r = (Math.random() * 16) | 0
      return r.toString(16)
    })
  }

  // 深拷贝一个对象
  function deepCopy (thing) {
    return JSON.parse(JSON.stringify(thing))
  }

  // 解析语义化版本号
  function parseSemVer (version) {
    var tmp = version.split('.')
    return {
      major: parseInt(tmp[0], 10),
      minor: parseInt(tmp[1], 10),
      patch: parseInt(tmp[2], 10)
    }
  }

  // 检查版本号是否符合最小要求
  // 返回 true 表示版本号大于等于最小要求
  function validSemanticVersion (version, minimum) {
    version = parseSemVer(version)
    minimum = parseSemVer(minimum)

    var versionNum = (version.major * 100000 * 100000) +
                     (version.minor * 100000) +
                     version.patch
    var minimumNum = (minimum.major * 100000 * 100000) +
                     (minimum.minor * 100000) +
                     minimum.patch

    return versionNum >= minimumNum
  }

  // 替换模板字符串中的变量
  function interpolateTemplate (str, obj) {
    for (var key in obj) {
      if (!obj.hasOwnProperty(key)) continue
      var keyTemplateStr = '{' + key + '}'
      var value = obj[key]
      while (str.indexOf(keyTemplateStr) !== -1) {
        str = str.replace(keyTemplateStr, value)
      }
    }
    return str
  }

  // 如果需要运行断言，则执行以下代码
  if (RUN_ASSERTS) {
  // 断言插值模板函数对于不同情况的输出是否符合预期
  console.assert(interpolateTemplate('abc', {a: 'x'}) === 'abc')
  console.assert(interpolateTemplate('{a}bc', {}) === '{a}bc')
  console.assert(interpolateTemplate('{a}bc', {p: 'q'}) === '{a}bc')
  console.assert(interpolateTemplate('{a}bc', {a: 'x'}) === 'xbc')
  console.assert(interpolateTemplate('{a}bc{a}bc', {a: 'x'}) === 'xbcxbc')
  console.assert(interpolateTemplate('{a}{a}{b}', {a: 'x', b: 'y'}) === 'xxy')
}

// ---------------------------------------------------------------------------
// Predicates
// ---------------------------------------------------------------------------

// 判断是否为字符串
function isString (s) {
  return typeof s === 'string'
}

// 判断是否为函数
function isFunction (f) {
  return typeof f === 'function'
}

// 判断是否为整数
function isInteger (n) {
  return typeof n === 'number' &&
         isFinite(n) &&
         Math.floor(n) === n
}

// 验证动画速度是否合法
function validAnimationSpeed (speed) {
  if (speed === 'fast' || speed === 'slow') return true
  if (!isInteger(speed)) return false
  return speed >= 0
}

// 验证节流速率是否合法
function validThrottleRate (rate) {
  return isInteger(rate) &&
         rate >= 1
}

// 验证棋子移动是否合法
function validMove (move) {
  // 移动应为字符串
  if (!isString(move)) return false

  // 移动应为形如 "e2-e4", "f6-d5" 的格式
  var squares = move.split('-')
  if (squares.length !== 2) return false

  return validSquare(squares[0]) && validSquare(squares[1])
}

// 验证棋盘位置是否合法
function validSquare (square) {
  return isString(square) && square.search(/^[a-h][1-8]$/) !== -1
}

// 如果运行断言，进行相关验证
if (RUN_ASSERTS) {
  console.assert(validSquare('a1'))
  console.assert(validSquare('e2'))
  console.assert(!validSquare('D2'))
  console.assert(!validSquare('g9'))
  console.assert(!validSquare('a'))
  console.assert(!validSquare(true))
  console.assert(!validSquare(null))
  console.assert(!validSquare({}))
}

// 验证棋子代码是否合法
function validPieceCode (code) {
  return isString(code) && code.search(/^[bw][KQRNBP]$/) !== -1
}

// 如果运行断言，进行相关验证
if (RUN_ASSERTS) {
  // 断言给定的棋子代码是否有效
  console.assert(validPieceCode('bP'))
  console.assert(validPieceCode('bK'))
  console.assert(validPieceCode('wK'))
  console.assert(validPieceCode('wR'))
  console.assert(!validPieceCode('WR'))
  console.assert(!validPieceCode('Wr'))
  console.assert(!validPieceCode('a'))
  console.assert(!validPieceCode(true))
  console.assert(!validPieceCode(null))
  console.assert(!validPieceCode({}))

  // 验证 FEN 格式是否有效
  function validFen (fen) {
    if (!isString(fen)) return false

    // 从末尾截取任何移动、王车易位等信息，只关注位置信息
    fen = fen.replace(/ .+$/, '')

    // 将空白格子的数字扩展为 1
    fen = expandFenEmptySquares(fen)

    // FEN 应该由 8 个由斜杠分隔的部分组成
    var chunks = fen.split('/')
    if (chunks.length !== 8) return false

    // 检查每个部分
    for (var i = 0; i < 8; i++) {
      if (chunks[i].length !== 8 ||
          chunks[i].search(/[^kqrnbpKQRNBP1]/) !== -1) {
        return false
      }
    }

    return true
  }

  // 如果运行断言，则进行断言测试
  if (RUN_ASSERTS) {
    console.assert(validFen(START_FEN))
    console.assert(validFen('8/8/8/8/8/8/8/8'))
    console.assert(validFen('r1bqkbnr/pppp1ppp/2n5/1B2p3/4P3/5N2/PPPP1PPP/RNBQK2R'))
    console.assert(validFen('3r3r/1p4pp/2nb1k2/pP3p2/8/PB2PN2/p4PPP/R4RK1 b - - 0 1'))
    console.assert(!validFen('3r3z/1p4pp/2nb1k2/pP3p2/8/PB2PN2/p4PPP/R4RK1 b - - 0 1'))
    console.assert(!validFen('anbqkbnr/8/8/8/8/8/PPPPPPPP/8'))
    console.assert(!validFen('rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/'))
    console.assert(!validFen('rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBN'))
    console.assert(!validFen('888888/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR'))
    console.assert(!validFen('888888/pppppppp/74/8/8/8/8/PPPPPPPP/RNBQKBNR'))
    console.assert(!validFen({}))
  }

  // 验证位置对象是否有效
  function validPositionObject (pos) {
    if (!$.isPlainObject(pos)) return false
  // 遍历位置对象中的每个属性
  for (var i in pos) {
    // 如果属性不是对象自身的属性，则跳过
    if (!pos.hasOwnProperty(i)) continue

    // 如果位置不合法或者棋子代码不合法，则返回 false
    if (!validSquare(i) || !validPieceCode(pos[i])) {
      return false
    }
  }

  // 返回 true
  return true
}

// 如果运行断言，则进行断言检查
if (RUN_ASSERTS) {
  console.assert(validPositionObject(START_POSITION))
  console.assert(validPositionObject({}))
  console.assert(validPositionObject({e2: 'wP'}))
  console.assert(validPositionObject({e2: 'wP', d2: 'wP'}))
  console.assert(!validPositionObject({e2: 'BP'}))
  console.assert(!validPositionObject({y2: 'wP'}))
  console.assert(!validPositionObject(null))
  console.assert(!validPositionObject('start'))
  console.assert(!validPositionObject(START_FEN))
}

// 检测是否为触摸设备
function isTouchDevice () {
  return 'ontouchstart' in document.documentElement
}

// 检测 jQuery 版本是否合法
function validJQueryVersion () {
  return typeof window.$ &&
         $.fn &&
         $.fn.jquery &&
         validSemanticVersion($.fn.jquery, MINIMUM_JQUERY_VERSION)
}

// ---------------------------------------------------------------------------
// Chess Util Functions
// ---------------------------------------------------------------------------

// 将 FEN 棋子代码转换为 bP、wK 等形式
function fenToPieceCode (piece) {
  // 黑色棋子
  if (piece.toLowerCase() === piece) {
    return 'b' + piece.toUpperCase()
  }

  // 白色棋子
  return 'w' + piece.toUpperCase()
}

// 将 bP、wK 等代码转换为 FEN 结构
function pieceCodeToFen (piece) {
  var pieceCodeLetters = piece.split('')

  // 白色棋子
  if (pieceCodeLetters[0] === 'w') {
    return pieceCodeLetters[1].toUpperCase()
  }

  // 黑色棋子
  return pieceCodeLetters[1].toLowerCase()
}

// 将 FEN 字符串转换为位置对象
// 如果 FEN 字符串无效，则返回 false
function fenToObj (fen) {
  if (!validFen(fen)) return false

  // 去除末尾的任何移动、王车易位等信息
  // 我们只关心位置信息
  fen = fen.replace(/ .+$/, '')
}
    // 将 FEN 字符串按 '/' 分割成行数组
    var rows = fen.split('/')
    // 创建一个空对象用于存储棋盘位置信息
    var position = {}
    
    // 初始化当前行号为 8
    var currentRow = 8
    for (var i = 0; i < 8; i++) {
      // 将当前行按字符分割成数组
      var row = rows[i].split('')
      var colIdx = 0
    
      // 遍历 FEN 每个部分的字符
      for (var j = 0; j < row.length; j++) {
        // 数字表示空格
        if (row[j].search(/[1-8]/) !== -1) {
          // 将数字字符转换为数字，表示空格数
          var numEmptySquares = parseInt(row[j], 10)
          colIdx = colIdx + numEmptySquares
        } else {
          // 棋子
          var square = COLUMNS[colIdx] + currentRow
          // 将棋子位置和对应的棋子代码存入位置对象
          position[square] = fenToPieceCode(row[j])
          colIdx = colIdx + 1
        }
      }
    
      currentRow = currentRow - 1
    }
    
    // 返回存储棋盘位置信息的对象
    return position
    }
    
    // 将位置对象转换为 FEN 字符串
    // 如果对象不是有效的位置对象，则返回 false
    function objToFen (obj) {
      // 如果对象不是有效的位置对象，则返回 false
      if (!validPositionObject(obj)) return false
    
      var fen = ''
    
      var currentRow = 8
      for (var i = 0; i < 8; i++) {
        for (var j = 0; j < 8; j++) {
          var square = COLUMNS[j] + currentRow
    
          // 如果位置对象中存在棋子
          if (obj.hasOwnProperty(square)) {
            // 将棋子代码转换为 FEN 格式并添加到字符串中
            fen = fen + pieceCodeToFen(obj[square])
          } else {
            // 空格
            fen = fen + '1'
          }
        }
    
        // 每行结束添加 '/'
        if (i !== 7) {
          fen = fen + '/'
        }
    
        currentRow = currentRow - 1
      }
    
      // 压缩连续的空格数
      fen = squeezeFenEmptySquares(fen)
    
      return fen
    }
    
    // 运行断言检查
    if (RUN_ASSERTS) {
      console.assert(objToFen(START_POSITION) === START_FEN)
      console.assert(objToFen({}) === '8/8/8/8/8/8/8/8')
      console.assert(objToFen({a2: 'wP', 'b2': 'bP'}) === '8/8/8/8/8/8/Pp6/8')
    }
    
    // 压缩 FEN 字符串中的连续空格数
    function squeezeFenEmptySquares (fen) {
      return fen.replace(/11111111/g, '8')
        .replace(/1111111/g, '7')
        .replace(/111111/g, '6')
        .replace(/11111/g, '5')
        .replace(/1111/g, '4')
        .replace(/111/g, '3')
        .replace(/11/g, '2')
    }
    
    function expandFenEmptySquares (fen) {
    // 将字符串中的 '8' 替换为 '11111111'，依次类推替换 '7'、'6'、'5'、'4'、'3'、'2'
    return fen.replace(/8/g, '11111111')
      .replace(/7/g, '1111111')
      .replace(/6/g, '111111')
      .replace(/5/g, '11111')
      .replace(/4/g, '1111')
      .replace(/3/g, '111')
      .replace(/2/g, '11')
    }
    
    // 返回两个方块之间的距离
    function squareDistance (squareA, squareB) {
      // 将方块坐标转换为数组形式
      var squareAArray = squareA.split('')
      var squareAx = COLUMNS.indexOf(squareAArray[0]) + 1
      var squareAy = parseInt(squareAArray[1], 10)
    
      var squareBArray = squareB.split('')
      var squareBx = COLUMNS.indexOf(squareBArray[0]) + 1
      var squareBy = parseInt(squareBArray[1], 10)
    
      // 计算 x 轴和 y 轴的距离
      var xDelta = Math.abs(squareAx - squareBx)
      var yDelta = Math.abs(squareAy - squareBy)
    
      // 返回较大的距离
      if (xDelta >= yDelta) return xDelta
      return yDelta
    }
    
    // 返回最接近方块的棋子的方块
    // 如果在位置中找不到棋子的实例，则返回 false
    function findClosestPiece (position, piece, square) {
      // 创建从方块开始的最近方块数组
      var closestSquares = createRadius(square)
    
      // 按距离顺序搜索棋子在位置中的位置
      for (var i = 0; i < closestSquares.length; i++) {
        var s = closestSquares[i]
    
        if (position.hasOwnProperty(s) && position[s] === piece) {
          return s
        }
      }
    
      return false
    }
    
    // 返回从方块开始的最近方块数组
    function createRadius (square) {
      var squares = []
    
      // 计算所有方块的距离
      for (var i = 0; i < 8; i++) {
        for (var j = 0; j < 8; j++) {
          var s = COLUMNS[i] + (j + 1)
    
          // 跳过起始方块
          if (square === s) continue
    
          squares.push({
            square: s,
            distance: squareDistance(square, s)
          })
        }
      }
    
      // 按距离排序
      squares.sort(function (a, b) {
        return a.distance - b.distance
      })
    
      // 只返回方块代码
      var surroundingSquares = []
    // 遍历 squares 数组，将每个元素的 square 属性添加到 surroundingSquares 数组中
    for (i = 0; i < squares.length; i++) {
      surroundingSquares.push(squares[i].square)
    }

    // 返回包含周围方块的数组
    return surroundingSquares
  }

  // 给定一个位置和一组移动，返回执行移动后的新位置
  function calculatePositionFromMoves (position, moves) {
    // 深拷贝当前位置，得到新的位置对象
    var newPosition = deepCopy(position)

    // 遍历移动数组
    for (var i in moves) {
      if (!moves.hasOwnProperty(i)) continue

      // 如果当前位置没有源方块上的棋子，则跳过该移动
      if (!newPosition.hasOwnProperty(i)) continue

      // 获取源方块上的棋子，删除源方块上的棋子，将棋子移动到目标方块
      var piece = newPosition[i]
      delete newPosition[i]
      newPosition[moves[i]] = piece
    }

    // 返回新的位置对象
    return newPosition
  }

  // TODO: 在 calculatePositionFromMoves 函数中添加一些断言

  // ---------------------------------------------------------------------------
  // HTML
  // ---------------------------------------------------------------------------

  // 构建包含棋盘和可能的多余棋子的 HTML 结构
  function buildContainerHTML (hasSparePieces) {
    var html = '<div class="{chessboard}">'

    // 如果有多余棋子，添加顶部多余棋子容器
    if (hasSparePieces) {
      html += '<div class="{sparePieces} {sparePiecesTop}"></div>'
    }

    // 添加棋盘容器
    html += '<div class="{board}"></div>'

    // 如果有多余棋子，添加底部多余棋子容器
    if (hasSparePieces) {
      html += '<div class="{sparePieces} {sparePiecesBottom}"></div>'
    }

    // 结束整个容器的 HTML 结构
    html += '</div>'

    // 使用 CSS 模板插值生成最终的 HTML 结构
    return interpolateTemplate(html, CSS)
  }

  // ---------------------------------------------------------------------------
  // Config
  // ---------------------------------------------------------------------------

  // 扩展配置参数的简写形式
  function expandConfigArgumentShorthand (config) {
    // 如果配置参数是 'start'，则将其转换为包含起始位置的配置对象
    if (config === 'start') {
      config = {position: deepCopy(START_POSITION)}
    } else if (validFen(config)) {
      // 如果配置参数是有效的 FEN 字符串，则将其转换为包含对应位置的配置对象
      config = {position: fenToObj(config)}
    } else if (validPositionObject(config)) {
      // 如果配置参数是有效的位置对象，则深拷贝该对象作为新的配置对象
      config = {position: deepCopy(config)}
    }

    // 确保配置参数是对象类型，如果不是则设置为空对象
    if (!$.isPlainObject(config)) config = {}

    // 返回扩展后的配置对象
    return config
  }

  // 验证配置参数并设置默认选项
  function expandConfig (config) {
    // 如果配置中的方向不是黑色，则将其设置为白色
    if (config.orientation !== 'black') config.orientation = 'white'

    // 如果配置中的显示注释不是false，则将其设置为true
    if (config.showNotation !== false) config.showNotation = true

    // 如果配置中的可拖动性不是true，则将其设置为false
    if (config.draggable !== true) config.draggable = false

    // 如果配置中的离开棋盘方式不是'trash'，则将其设置为'snapback'
    if (config.dropOffBoard !== 'trash') config.dropOffBoard = 'snapback'

    // 如果配置中的备用棋子不是true，则将其设置为false
    if (config.sparePieces !== true) config.sparePieces = false

    // 如果启用备用棋子，则必须将可拖动性设置为true
    if (config.sparePieces) config.draggable = true

    // 默认棋子主题是维基百科
    if (!config.hasOwnProperty('pieceTheme') ||
        (!isString(config.pieceTheme) && !isFunction(config.pieceTheme))) {
      config.pieceTheme = 'img/chesspieces/wikipedia/{piece}.png'
    }

    // 动画速度
    if (!validAnimationSpeed(config.appearSpeed)) config.appearSpeed = DEFAULT_APPEAR_SPEED
    if (!validAnimationSpeed(config.moveSpeed)) config.moveSpeed = DEFAULT_MOVE_SPEED
    if (!validAnimationSpeed(config.snapbackSpeed)) config.snapbackSpeed = DEFAULT_SNAPBACK_SPEED
    if (!validAnimationSpeed(config.snapSpeed)) config.snapSpeed = DEFAULT_SNAP_SPEED
    if (!validAnimationSpeed(config.trashSpeed)) config.trashSpeed = DEFAULT_TRASH_SPEED

    // 节流速率
    if (!validThrottleRate(config.dragThrottleRate)) config.dragThrottleRate = DEFAULT_DRAG_THROTTLE_RATE

    // 返回配置
    return config
  }

  // ---------------------------------------------------------------------------
  // 依赖项
  // ---------------------------------------------------------------------------

  // 检查是否有兼容版本的jQuery
  function checkJQuery () {
    // 检查是否存在有效的 jQuery 版本，如果不存在则弹出错误信息并返回 false
    if (!validJQueryVersion()) {
      var errorMsg = 'Chessboard Error 1005: Unable to find a valid version of jQuery. ' +
        'Please include jQuery ' + MINIMUM_JQUERY_VERSION + ' or higher on the page' +
        '\n\n' +
        'Exiting' + ELLIPSIS
      window.alert(errorMsg)
      return false
    }

    // 返回 true
    return true
  }

  // 返回布尔值 false 或者 $container 元素
  function checkContainerArg (containerElOrString) {
    // 如果 containerElOrString 为空字符串，则弹出错误信息并返回 false
    if (containerElOrString === '') {
      var errorMsg1 = 'Chessboard Error 1001: ' +
        'The first argument to Chessboard() cannot be an empty string.' +
        '\n\n' +
        'Exiting' + ELLIPSIS
      window.alert(errorMsg1)
      return false
    }

    // 如果 containerElOrString 是字符串且不以 '#' 开头，则转换为 jQuery 选择器
    if (isString(containerElOrString) &&
        containerElOrString.charAt(0) !== '#') {
      containerElOrString = '#' + containerElOrString
    }

    // 将 containerEl 转换为 jQuery 集合，必须是大小为 1 的集合
    var $container = $(containerElOrString)
    if ($container.length !== 1) {
      var errorMsg2 = 'Chessboard Error 1003: ' +
        'The first argument to Chessboard() must be the ID of a DOM node, ' +
        'an ID query selector, or a single DOM node.' +
        '\n\n' +
        'Exiting' + ELLIPSIS
      window.alert(errorMsg2)
      return false
    }

    // 返回 $container
    return $container
  }

  // ---------------------------------------------------------------------------
  // Constructor
  // ---------------------------------------------------------------------------

  // 构造函数
  function constructor (containerElOrString, config) {
    // 首先检查基本依赖项
    if (!checkJQuery()) return null
    var $container = checkContainerArg(containerElOrString)
    if (!$container) return null

    // 确保 config 对象符合预期
    config = expandConfigArgumentShorthand(config)
    config = expandConfig(config)

    // DOM 元素
    var $board = null
    // 初始化拖动的棋子对象
    var $draggedPiece = null
    // 顶部备用棋子容器对象
    var $sparePiecesTop = null
    // 底部备用棋子容器对象
    var $sparePiecesBottom = null

    // 构造函数返回的对象
    var widget = {}

    // -------------------------------------------------------------------------
    // Stateful
    // -------------------------------------------------------------------------

    // 棋盘边框大小
    var boardBorderSize = 2
    // 当前棋盘方向
    var currentOrientation = 'white'
    // 当前棋盘位置
    var currentPosition = {}
    // 正在拖动的棋子对象
    var draggedPiece = null
    // 拖动的棋子位置
    var draggedPieceLocation = null
    // 拖动的棋子来源
    var draggedPieceSource = null
    // 是否正在拖动棋子
    var isDragging = false
    // 备用棋子元素的 ID
    var sparePiecesElsIds = {}
    // 棋盘方格元素的 ID
    var squareElsIds = {}
    // 棋盘方格的偏移量
    var squareElsOffsets = {}
    // 棋盘方格大小
    var squareSize = 16

    // -------------------------------------------------------------------------
    // Validation / Errors
    // -------------------------------------------------------------------------

    // 错误处理函数
    function error (code, msg, obj) {
      // 如果未设置 showErrors，则不执行任何操作
      if (
        config.hasOwnProperty('showErrors') !== true ||
          config.showErrors === false
      ) {
        return
      }

      var errorText = 'Chessboard Error ' + code + ': ' + msg

      // 打印到控制台
      if (
        config.showErrors === 'console' &&
          typeof console === 'object' &&
          typeof console.log === 'function'
      ) {
        console.log(errorText)
        if (arguments.length >= 2) {
          console.log(obj)
        }
        return
      }

      // 弹出错误提示框
      if (config.showErrors === 'alert') {
        if (obj) {
          errorText += '\n\n' + JSON.stringify(obj)
        }
        window.alert(errorText)
        return
      }

      // 自定义错误处理函数
      if (isFunction(config.showErrors)) {
        config.showErrors(code, msg, obj)
      }
    }
    // 设置初始状态函数
    function setInitialState () {
      // 将当前方向设置为配置文件中的方向
      currentOrientation = config.orientation

      // 确保位置是有效的
      if (config.hasOwnProperty('position')) {
        // 如果配置中的位置为 'start'，则将当前位置设置为起始位置的深拷贝
        if (config.position === 'start') {
          currentPosition = deepCopy(START_POSITION)
        } 
        // 如果配置中的位置是有效的 FEN 格式，则将当前位置设置为对应的对象
        else if (validFen(config.position)) {
          currentPosition = fenToObj(config.position)
        } 
        // 如果配置中的位置是有效的位置对象，则将当前位置设置为配置中位置的深拷贝
        else if (validPositionObject(config.position)) {
          currentPosition = deepCopy(config.position)
        } 
        // 如果配置中的位置无效，则输出错误信息
        else {
          error(
            7263,
            'Invalid value passed to config.position.',
            config.position
          )
        }
      }
    }

    // -------------------------------------------------------------------------
    // DOM Misc
    // -------------------------------------------------------------------------

    // 根据容器的宽度计算棋盘格子的大小
    // 这里有一些 CSS 的黑魔法，让我来解释一下：
    // 获取容器元素的宽度（可以是任何东西），减去 1 作为修正因子，然后不断减小直到找到一个能被 8 整除的确切值作为格子大小
    function calculateSquareSize () {
      var containerWidth = parseInt($container.width(), 10)

      // 防御性编程，防止无限循环
      if (!containerWidth || containerWidth <= 0) {
        return 0
      }

      // 加一个像素的填充
      var boardWidth = containerWidth - 1

      // 不断减小宽度直到能被 8 整除
      while (boardWidth % 8 !== 0 && boardWidth > 0) {
        boardWidth = boardWidth - 1
      }

      return boardWidth / 8
    }

    // 为元素创建随机 ID
    // 创建棋盘格子的唯一标识符
    function createElIds () {
      // 棋盘上的方块
      for (var i = 0; i < COLUMNS.length; i++) {
        for (var j = 1; j <= 8; j++) {
          var square = COLUMNS[i] + j
          squareElsIds[square] = square + '-' + uuid()
        }
      }

      // 备用棋子
      var pieces = 'KQRNBP'.split('')
      for (i = 0; i < pieces.length; i++) {
        var whitePiece = 'w' + pieces[i]
        var blackPiece = 'b' + pieces[i]
        sparePiecesElsIds[whitePiece] = whitePiece + '-' + uuid()
        sparePiecesElsIds[blackPiece] = blackPiece + '-' + uuid()
      }
    }

    // -------------------------------------------------------------------------
    // Markup Building
    // -------------------------------------------------------------------------
    // 构建棋盘的 HTML 结构
    function buildBoardHTML (orientation) {
      // 如果方向不是黑色，则设置为白色
      if (orientation !== 'black') {
        orientation = 'white'
      }

      var html = ''

      // 代数标记 / 方向
      var alpha = deepCopy(COLUMNS)
      var row = 8
      // 如果方向是黑色，则反转字母数组并设置行数为1
      if (orientation === 'black') {
        alpha.reverse()
        row = 1
      }

      var squareColor = 'white'
      for (var i = 0; i < 8; i++) {
        html += '<div class="{row}">'
        for (var j = 0; j < 8; j++) {
          var square = alpha[j] + row

          html += '<div class="{square} ' + CSS[squareColor] + ' ' +
            'square-' + square + '" ' +
            'style="width:' + squareSize + 'px;height:' + squareSize + 'px;" ' +
            'id="' + squareElsIds[square] + '" ' +
            'data-square="' + square + '">'

          if (config.showNotation) {
            // 字母标记
            if ((orientation === 'white' && row === 1) ||
                (orientation === 'black' && row === 8)) {
              html += '<div class="{notation} {alpha}">' + alpha[j] + '</div>'
            }

            // 数字标记
            if (j === 0) {
              html += '<div class="{notation} {numeric}">' + row + '</div>'
            }
          }

          html += '</div>' // 结束 .square

          squareColor = (squareColor === 'white') ? 'black' : 'white'
        }
        html += '<div class="{clearfix}"></div></div>'

        squareColor = (squareColor === 'white') ? 'black' : 'white'

        // 根据方向更新行数
        if (orientation === 'white') {
          row = row - 1
        } else {
          row = row + 1
        }
      }

      // 使用 CSS 样式插值模板返回最终的 HTML 结构
      return interpolateTemplate(html, CSS)
    }
    // 构建棋子图片的源路径
    function buildPieceImgSrc (piece) {
      // 如果配置中的棋子主题是函数，则调用该函数返回结果
      if (isFunction(config.pieceTheme)) {
        return config.pieceTheme(piece)
      }

      // 如果配置中的棋子主题是字符串，则使用插值模板替换变量返回结果
      if (isString(config.pieceTheme)) {
        return interpolateTemplate(config.pieceTheme, {piece: piece})
      }

      // 如果出现此情况，应该是不可能发生的，输出错误信息并返回空字符串
      error(8272, 'Unable to build image source for config.pieceTheme.')
      return ''
    }

    // 构建棋子的 HTML 元素
    function buildPieceHTML (piece, hidden, id) {
      // 初始化 HTML 字符串
      var html = '<img src="' + buildPieceImgSrc(piece) + '" '
      // 如果传入了 ID 参数且不为空，则添加 ID 属性
      if (isString(id) && id !== '') {
        html += 'id="' + id + '" '
      }
      // 添加其他属性和样式
      html += 'alt="" ' +
        'class="{piece}" ' +
        'data-piece="' + piece + '" ' +
        'style="width:' + squareSize + 'px;' + 'height:' + squareSize + 'px;'

      // 如果隐藏标志为真，则添加 display:none 样式
      if (hidden) {
        html += 'display:none;'
      }

      // 结束标签
      html += '" />'

      // 使用 CSS 模板插值替换 HTML 字符串中的变量并返回结果
      return interpolateTemplate(html, CSS)
    }

    // 构建备用棋子的 HTML 元素
    function buildSparePiecesHTML (color) {
      // 初始化棋子数组
      var pieces = ['wK', 'wQ', 'wR', 'wB', 'wN', 'wP']
      // 如果颜色为黑色，则使用黑色棋子数组
      if (color === 'black') {
        pieces = ['bK', 'bQ', 'bR', 'bB', 'bN', 'bP']
      }

      // 初始化 HTML 字符串
      var html = ''
      // 遍历棋子数组，构建每个棋子的 HTML 元素并拼接到 HTML 字符串中
      for (var i = 0; i < pieces.length; i++) {
        html += buildPieceHTML(pieces[i], false, sparePiecesElsIds[pieces[i]])
      }

      // 返回构建好的 HTML 字符串
      return html
    }

    // -------------------------------------------------------------------------
    // Animations
    // -------------------------------------------------------------------------
    // 定义一个函数，用于将棋子从一个方块移动到另一个方块，并执行完成后的回调函数
    function animateSquareToSquare (src, dest, piece, completeFn) {
      // 获取源方块和目标方块的信息
      var $srcSquare = $('#' + squareElsIds[src])
      var srcSquarePosition = $srcSquare.offset()
      var $destSquare = $('#' + squareElsIds[dest])
      var destSquarePosition = $destSquare.offset()

      // 创建动画棋子并绝对定位在源方块上
      var animatedPieceId = uuid()
      $('body').append(buildPieceHTML(piece, true, animatedPieceId))
      var $animatedPiece = $('#' + animatedPieceId)
      $animatedPiece.css({
        display: '',
        position: 'absolute',
        top: srcSquarePosition.top,
        left: srcSquarePosition.left
      })

      // 从源方块中移除原始棋子
      $srcSquare.find('.' + CSS.piece).remove()

      function onFinishAnimation1 () {
        // 将“真实”棋子添加到目标方块
        $destSquare.append(buildPieceHTML(piece))

        // 移除动画棋子
        $animatedPiece.remove()

        // 执行完成函数
        if (isFunction(completeFn)) {
          completeFn()
        }
      }

      // 将棋子动画到目标方块
      var opts = {
        duration: config.moveSpeed,
        complete: onFinishAnimation1
      }
      $animatedPiece.animate(destSquarePosition, opts)
    }
    // 将一个棋子动画移动到目标位置
    function animateSparePieceToSquare (piece, dest, completeFn) {
      // 获取起始位置的偏移量
      var srcOffset = $('#' + sparePiecesElsIds[piece]).offset()
      // 获取目标位置的元素
      var $destSquare = $('#' + squareElsIds[dest])
      // 获取目标位置的偏移量
      var destOffset = $destSquare.offset()

      // 创建动画棋子
      var pieceId = uuid()
      $('body').append(buildPieceHTML(piece, true, pieceId))
      var $animatedPiece = $('#' + pieceId)
      $animatedPiece.css({
        display: '',
        position: 'absolute',
        left: srcOffset.left,
        top: srcOffset.top
      })

      // 动画完成时的操作
      function onFinishAnimation2 () {
        // 将“真实”棋子添加到目标位置
        $destSquare.find('.' + CSS.piece).remove()
        $destSquare.append(buildPieceHTML(piece))

        // 移除动画棋子
        $animatedPiece.remove()

        // 运行完成函数
        if (isFunction(completeFn)) {
          completeFn()
        }
      }

      // 将棋子动画移动到目标位置
      var opts = {
        duration: config.moveSpeed,
        complete: onFinishAnimation2
      }
      $animatedPiece.animate(destOffset, opts)
    }

    // 执行一个动画数组
    // 执行动画效果，根据动画数组中的内容移动棋子
    function doAnimations (animations, oldPos, newPos) {
      // 如果没有动画，直接返回
      if (animations.length === 0) return

      // 记录已完成的动画数量
      var numFinished = 0
      // 完成动画时的回调函数
      function onFinishAnimation3 () {
        // 如果还有未完成的动画，直接退出
        numFinished = numFinished + 1
        if (numFinished !== animations.length) return

        // 绘制最终位置
        drawPositionInstant()

        // 运行 onMoveEnd 函数
        if (isFunction(config.onMoveEnd)) {
          config.onMoveEnd(deepCopy(oldPos), deepCopy(newPos))
        }
      }

      // 遍历动画数组
      for (var i = 0; i < animations.length; i++) {
        var animation = animations[i]

        // 清除一个棋子
        if (animation.type === 'clear') {
          $('#' + squareElsIds[animation.square] + ' .' + CSS.piece)
            .fadeOut(config.trashSpeed, onFinishAnimation3)

        // 添加一个棋子，没有备用棋子 - 渐变显示棋子
        } else if (animation.type === 'add' && !config.sparePieces) {
          $('#' + squareElsIds[animation.square])
            .append(buildPieceHTML(animation.piece, true))
            .find('.' + CSS.piece)
            .fadeIn(config.appearSpeed, onFinishAnimation3)

        // 添加一个棋子，有备用棋子 - 从备用棋子处动画显示到目标位置
        } else if (animation.type === 'add' && config.sparePieces) {
          animateSparePieceToSquare(animation.piece, animation.square, onFinishAnimation3)

        // 移动一个棋子从 squareA 到 squareB
        } else if (animation.type === 'move') {
          animateSquareToSquare(animation.source, animation.destination, animation.piece, onFinishAnimation3)
        }
      }
    }

    // 计算从 pos1 到 pos2 需要进行的动画数组
    function calculateAnimations (pos1, pos2) {
      // 计算两个位置之间的动画效果
      // 复制两个位置对象
      pos1 = deepCopy(pos1)
      pos2 = deepCopy(pos2)
    
      var animations = []
      var squaresMovedTo = {}
    
      // 移除两个位置中相同的棋子
      for (var i in pos2) {
        if (!pos2.hasOwnProperty(i)) continue
    
        if (pos1.hasOwnProperty(i) && pos1[i] === pos2[i]) {
          delete pos1[i]
          delete pos2[i]
        }
      }
    
      // 查找所有的“移动”动画
      for (i in pos2) {
        if (!pos2.hasOwnProperty(i)) continue
    
        var closestPiece = findClosestPiece(pos1, pos2[i], i)
        if (closestPiece) {
          animations.push({
            type: 'move',
            source: closestPiece,
            destination: i,
            piece: pos2[i]
          })
    
          delete pos1[closestPiece]
          delete pos2[i]
          squaresMovedTo[i] = true
        }
      }
    
      // 添加“添加”动画
      for (i in pos2) {
        if (!pos2.hasOwnProperty(i)) continue
    
        animations.push({
          type: 'add',
          square: i,
          piece: pos2[i]
        })
    
        delete pos2[i]
      }
    
      // 添加“清除”动画
      for (i in pos1) {
        if (!pos1.hasOwnProperty(i)) continue
    
        // 如果棋子在一个被“移动”到的方格上，则不清除
        if (squaresMovedTo.hasOwnProperty(i)) continue
    
        animations.push({
          type: 'clear',
          square: i,
          piece: pos1[i]
        })
    
        delete pos1[i]
      }
    
      // 返回动画数组
      return animations
    }
    
    // -------------------------------------------------------------------------
    // Control Flow
    // -------------------------------------------------------------------------
    // 绘制当前棋局的位置
    function drawPositionInstant () {
      // 清空棋盘
      $board.find('.' + CSS.piece).remove()

      // 添加棋子
      for (var i in currentPosition) {
        if (!currentPosition.hasOwnProperty(i)) continue

        $('#' + squareElsIds[i]).append(buildPieceHTML(currentPosition[i]))
      }
    }

    // 绘制棋盘
    function drawBoard () {
      // 生成棋盘的 HTML 结构
      $board.html(buildBoardHTML(currentOrientation, squareSize, config.showNotation))
      // 绘制当前棋局的位置
      drawPositionInstant()

      // 如果有备用棋子
      if (config.sparePieces) {
        // 根据当前棋盘方向设置备用棋子的 HTML 结构
        if (currentOrientation === 'white') {
          $sparePiecesTop.html(buildSparePiecesHTML('black'))
          $sparePiecesBottom.html(buildSparePiecesHTML('white'))
        } else {
          $sparePiecesTop.html(buildSparePiecesHTML('white'))
          $sparePiecesBottom.html(buildSparePiecesHTML('black'))
        }
      }
    }

    // 设置当前棋局位置
    function setCurrentPosition (position) {
      var oldPos = deepCopy(currentPosition)
      var newPos = deepCopy(position)
      var oldFen = objToFen(oldPos)
      var newFen = objToFen(newPos)

      // 如果位置没有改变，则不做任何操作
      if (oldFen === newFen) return

      // 运行配置中的 onChange 函数
      if (isFunction(config.onChange)) {
        config.onChange(oldPos, newPos)
      }

      // 更新当前棋局位置
      currentPosition = position
    }

    // 判断坐标 (x, y) 是否在棋盘上
    function isXYOnSquare (x, y) {
      for (var i in squareElsOffsets) {
        if (!squareElsOffsets.hasOwnProperty(i)) continue

        var s = squareElsOffsets[i]
        if (x >= s.left &&
            x < s.left + squareSize &&
            y >= s.top &&
            y < s.top + squareSize) {
          return i
        }
      }

      return 'offboard'
    }

    // 记录每个方格的坐标到内存中
    function captureSquareOffsets () {
      squareElsOffsets = {}

      for (var i in squareElsIds) {
        if (!squareElsIds.hasOwnProperty(i)) continue

        squareElsOffsets[i] = $('#' + squareElsIds[i]).offset()
      }
    }
    // 移除棋盘上所有方块的高亮效果
    function removeSquareHighlights () {
      $board
        .find('.' + CSS.square)
        .removeClass(CSS.highlight1 + ' ' + CSS.highlight2)
    }

    // 将被拖动的棋子还原到原来的位置
    function snapbackDraggedPiece () {
      // 如果被拖动的棋子来自备用棋子，则直接删除
      if (draggedPieceSource === 'spare') {
        trashDraggedPiece()
        return
      }

      removeSquareHighlights()

      // 动画完成后执行的函数
      function complete () {
        drawPositionInstant()
        $draggedPiece.css('display', 'none')

        // 执行 onSnapbackEnd 函数
        if (isFunction(config.onSnapbackEnd)) {
          config.onSnapbackEnd(
            draggedPiece,
            draggedPieceSource,
            deepCopy(currentPosition),
            currentOrientation
          )
        }
      }

      // 获取被拖动棋子的原始位置
      var sourceSquarePosition = $('#' + squareElsIds[draggedPieceSource]).offset()

      // 将棋子动画移动到目标位置
      var opts = {
        duration: config.snapbackSpeed,
        complete: complete
      }
      $draggedPiece.animate(sourceSquarePosition, opts)

      // 设置状态为非拖动状态
      isDragging = false
    }

    // 删除被拖动的棋子
    function trashDraggedPiece () {
      removeSquareHighlights()

      // 删除原始位置的棋子
      var newPosition = deepCopy(currentPosition)
      delete newPosition[draggedPieceSource]
      setCurrentPosition(newPosition)

      // 重新绘制棋盘
      drawPositionInstant()

      // 隐藏被拖动的棋子
      $draggedPiece.fadeOut(config.trashSpeed)

      // 设置状态为非拖动状态
      isDragging = false
    }
    // 当棋子被放置在一个方格上时，执行以下操作
    function dropDraggedPieceOnSquare (square) {
      // 移除方格的高亮显示
      removeSquareHighlights()

      // 更新棋盘位置
      var newPosition = deepCopy(currentPosition)
      delete newPosition[draggedPieceSource]
      newPosition[square] = draggedPiece
      setCurrentPosition(newPosition)

      // 获取目标方格的位置信息
      var targetSquarePosition = $('#' + squareElsIds[square]).offset()

      // 动画完成后执行的操作
      function onAnimationComplete () {
        // 立即绘制棋盘位置
        drawPositionInstant()
        $draggedPiece.css('display', 'none')

        // 执行配置中的 onSnapEnd 函数
        if (isFunction(config.onSnapEnd)) {
          config.onSnapEnd(draggedPieceSource, square, draggedPiece)
        }
      }

      // 将棋子移动到目标方格
      var opts = {
        duration: config.snapSpeed,
        complete: onAnimationComplete
      }
      $draggedPiece.animate(targetSquarePosition, opts)

      // 设置状态为非拖拽状态
      isDragging = false
    }
    // 开始拖动棋子的函数
    function beginDraggingPiece (source, piece, x, y) {
      // 运行他们自定义的 onDragStart 函数
      // 他们自定义的 onDragStart 函数可以取消拖动开始
      if (isFunction(config.onDragStart) &&
          config.onDragStart(source, piece, deepCopy(currentPosition), currentOrientation) === false) {
        return
      }

      // 设置状态为正在拖动
      isDragging = true
      // 记录被拖动的棋子
      draggedPiece = piece
      // 记录被拖动的棋子来源
      draggedPieceSource = source

      // 如果棋子来自备用棋子区域，则位置为离开棋盘
      if (source === 'spare') {
        draggedPieceLocation = 'offboard'
      } else {
        draggedPieceLocation = source
      }

      // 记录所有棋格的 x, y 坐标
      captureSquareOffsets()

      // 创建被拖动的棋子
      $draggedPiece.attr('src', buildPieceImgSrc(piece)).css({
        display: '',
        position: 'absolute',
        left: x - squareSize / 2,
        top: y - squareSize / 2
      })

      if (source !== 'spare') {
        // 高亮来源棋格并隐藏棋子
        $('#' + squareElsIds[source])
          .addClass(CSS.highlight1)
          .find('.' + CSS.piece)
          .css('display', 'none')
      }
    }
    // 更新被拖动棋子的位置
    function updateDraggedPiece (x, y) {
      // 将被拖动的棋子放在鼠标光标上方
      $draggedPiece.css({
        left: x - squareSize / 2,
        top: y - squareSize / 2
      })

      // 获取位置
      var location = isXYOnSquare(x, y)

      // 如果位置没有改变，则不执行任何操作
      if (location === draggedPieceLocation) return

      // 移除之前方格的高亮
      if (validSquare(draggedPieceLocation)) {
        $('#' + squareElsIds[draggedPieceLocation]).removeClass(CSS.highlight2)
      }

      // 添加新方格的高亮
      if (validSquare(location)) {
        $('#' + squareElsIds[location]).addClass(CSS.highlight2)
      }

      // 运行 onDragMove 函数
      if (isFunction(config.onDragMove)) {
        config.onDragMove(
          location,
          draggedPieceLocation,
          draggedPieceSource,
          draggedPiece,
          deepCopy(currentPosition),
          currentOrientation
        )
      }

      // 更新状态
      draggedPieceLocation = location
    }

    // 清空棋盘
    widget.clear = function (useAnimation) {
      widget.position({}, useAnimation)
    }

    // 从页面中移除小部件
    widget.destroy = function () {
      // 移除标记
      $container.html('')
      $draggedPiece.remove()

      // 移除事件处理程序
      $container.unbind()
    }

    // 获取当前 FEN 表示
    widget.fen = function () {
      return widget.position('fen')
    }

    // 翻转棋盘方向
    widget.flip = function () {
      return widget.orientation('flip')
    }

    // 移动棋子
    // TODO: 这个方法应该接受可变数量的参数以及接受一个移动数组
    // 定义一个移动函数，用于移动棋子
    widget.move = function () {
      // 如果没有传入参数，则不做任何操作
      // TODO: 应该返回当前位置
      if (arguments.length === 0) return

      var useAnimation = true

      // 收集移动操作到一个对象中
      var moves = {}
      for (var i = 0; i < arguments.length; i++) {
        // 如果传入参数为false，则不使用动画效果
        if (arguments[i] === false) {
          useAnimation = false
          continue
        }

        // 跳过无效的参数
        if (!validMove(arguments[i])) {
          error(2826, 'Invalid move passed to the move method.', arguments[i])
          continue
        }

        var tmp = arguments[i].split('-')
        moves[tmp[0]] = tmp[1]
      }

      // 根据移动操作计算新的位置
      var newPos = calculatePositionFromMoves(currentPosition, moves)

      // 更新棋盘
      widget.position(newPos, useAnimation)

      // 返回新的位置对象
      return newPos
    }

    // 定义一个设置棋盘方向的函数
    widget.orientation = function (arg) {
      // 如果没有传入参数，则返回当前方向
      if (arguments.length === 0) {
        return currentOrientation
      }

      // 设置为白色或黑色
      if (arg === 'white' || arg === 'black') {
        currentOrientation = arg
        drawBoard()
        return currentOrientation
      }

      // 翻转方向
      if (arg === 'flip') {
        currentOrientation = currentOrientation === 'white' ? 'black' : 'white'
        drawBoard()
        return currentOrientation
      }

      error(5482, 'Invalid value passed to the orientation method.', arg)
    }
    // 定义 widget 对象的 position 方法，用于设置棋盘的位置
    widget.position = function (position, useAnimation) {
      // 如果没有参数，返回当前位置的深拷贝
      if (arguments.length === 0) {
        return deepCopy(currentPosition)
      }

      // 如果参数是字符串 'fen'，返回当前位置的 FEN 表示
      if (isString(position) && position.toLowerCase() === 'fen') {
        return objToFen(currentPosition)
      }

      // 如果参数是字符串 'start'，将位置设置为初始位置
      if (isString(position) && position.toLowerCase() === 'start') {
        position = deepCopy(START_POSITION)
      }

      // 如果参数是合法的 FEN 表示，将其转换为位置对象
      if (validFen(position)) {
        position = fenToObj(position)
      }

      // 验证位置对象的有效性
      if (!validPositionObject(position)) {
        error(6482, 'Invalid value passed to the position method.', position)
        return
      }

      // 默认开启动画效果
      if (useAnimation !== false) useAnimation = true

      if (useAnimation) {
        // 计算动画效果
        var animations = calculateAnimations(currentPosition, position)
        // 执行动画
        doAnimations(animations, currentPosition, position)

        // 设置新的位置
        setCurrentPosition(position)
      } else {
        // 立即更新位置
        setCurrentPosition(position)
        drawPositionInstant()
      }
    }

    // 定义 widget 对象的 resize 方法，用于调整棋盘大小
    widget.resize = function () {
      // 计算新的方块大小
      squareSize = calculateSquareSize()

      // 设置棋盘宽度
      $board.css('width', squareSize * 8 + 'px')

      // 设置拖动棋子大小
      $draggedPiece.css({
        height: squareSize,
        width: squareSize
      })

      // 备用棋子
      if (config.sparePieces) {
        $container
          .find('.' + CSS.sparePieces)
          .css('paddingLeft', squareSize + boardBorderSize + 'px')
      }

      // 重绘棋盘
      drawBoard()
    }

    // 设置初始位置
    widget.start = function (useAnimation) {
      widget.position('start', useAnimation)
    }
    // -------------------------------------------------------------------------
    // Browser Events
    // -------------------------------------------------------------------------

    // 阻止默认事件的触发
    function stopDefault (evt) {
      evt.preventDefault()
    }

    // 鼠标按下事件处理函数
    function mousedownSquare (evt) {
      // 如果不可拖动，则不执行任何操作
      if (!config.draggable) return

      // 如果该方格上没有棋子，则不执行任何操作
      var square = $(this).attr('data-square')
      if (!validSquare(square)) return
      if (!currentPosition.hasOwnProperty(square)) return

      // 开始拖动棋子
      beginDraggingPiece(square, currentPosition[square], evt.pageX, evt.pageY)
    }

    // 触摸开始事件处理函数
    function touchstartSquare (e) {
      // 如果不可拖动，则不执行任何操作
      if (!config.draggable) return

      // 如果该方格上没有棋子，则不执行任何操作
      var square = $(this).attr('data-square')
      if (!validSquare(square)) return
      if (!currentPosition.hasOwnProperty(square)) return

      e = e.originalEvent
      // 开始拖动棋子
      beginDraggingPiece(
        square,
        currentPosition[square],
        e.changedTouches[0].pageX,
        e.changedTouches[0].pageY
      )
    }

    // 鼠标按下空白棋子事件处理函数
    function mousedownSparePiece (evt) {
      // 如果未启用备用棋子，则不执行任何操作
      if (!config.sparePieces) return

      var piece = $(this).attr('data-piece')

      // 开始拖动棋子
      beginDraggingPiece('spare', piece, evt.pageX, evt.pageY)
    }

    // 触摸开始空白棋子事件处理函数
    function touchstartSparePiece (e) {
      // 如果未启用备用棋子，则不执行任何操作
      if (!config.sparePieces) return

      var piece = $(this).attr('data-piece')

      e = e.originalEvent
      // 开始拖动棋子
      beginDraggingPiece(
        'spare',
        piece,
        e.changedTouches[0].pageX,
        e.changedTouches[0].pageY
      )
    }

    // 鼠标移动窗口事件处理函数
    function mousemoveWindow (evt) {
      // 如果正在拖动棋子，则更新拖动棋子的位置
      if (isDragging) {
        updateDraggedPiece(evt.pageX, evt.pageY)
      }
    }

    // 节流鼠标移动窗口事件处理函数
    var throttledMousemoveWindow = throttle(mousemoveWindow, config.dragThrottleRate)
    // 当触发 touchmove 事件时执行的函数
    function touchmoveWindow (evt) {
      // 如果没有拖动棋子，则不执行任何操作
      if (!isDragging) return

      // 阻止屏幕滚动
      evt.preventDefault()

      // 更新被拖动棋子的位置
      updateDraggedPiece(evt.originalEvent.changedTouches[0].pageX,
        evt.originalEvent.changedTouches[0].pageY)
    }

    // 创建一个节流函数，用于限制 touchmoveWindow 函数的执行频率
    var throttledTouchmoveWindow = throttle(touchmoveWindow, config.dragThrottleRate)

    // 当触发 mouseup 事件时执行的函数
    function mouseupWindow (evt) {
      // 如果没有拖动棋子，则不执行任何操作
      if (!isDragging) return

      // 获取位置信息
      var location = isXYOnSquare(evt.pageX, evt.pageY)

      // 停止拖动棋子
      stopDraggedPiece(location)
    }

    // 当触发 touchend 事件时执行的函数
    function touchendWindow (evt) {
      // 如果没有拖动棋子，则不执行任何操作
      if (!isDragging) return

      // 获取位置信息
      var location = isXYOnSquare(evt.originalEvent.changedTouches[0].pageX,
        evt.originalEvent.changedTouches[0].pageY)

      // 停止拖动棋子
      stopDraggedPiece(location)
    }

    // 当鼠标移入方块时执行的函数
    function mouseenterSquare (evt) {
      // 如果正在拖动棋子，则不触发此事件
      // 注意：这种情况不应该发生，但这是一种保护措施
      if (isDragging) return

      // 如果没有提供 onMouseoverSquare 函数，则退出
      if (!isFunction(config.onMouseoverSquare)) return

      // 获取方块信息
      var square = $(evt.currentTarget).attr('data-square')

      // 注意：这种情况不应该发生；防御性编程
      if (!validSquare(square)) return

      // 获取该方块上的棋子
      var piece = false
      if (currentPosition.hasOwnProperty(square)) {
        piece = currentPosition[square]
      }

      // 执行用户定义的函数
      config.onMouseoverSquare(square, piece, deepCopy(currentPosition), currentOrientation)
    }
    // 当鼠标移出方块时触发的事件处理函数
    function mouseleaveSquare (evt) {
      // 如果正在拖动棋子，则不触发此事件
      // 注意：这不应该发生，但这是一种保护措施
      if (isDragging) return

      // 如果未提供 onMouseoutSquare 函数，则退出
      if (!isFunction(config.onMouseoutSquare)) return

      // 获取方块
      var square = $(evt.currentTarget).attr('data-square')

      // 注意：这不应该发生；防御性编程
      if (!validSquare(square)) return

      // 获取该方块上的棋子
      var piece = false
      if (currentPosition.hasOwnProperty(square)) {
        piece = currentPosition[square]
      }

      // 执行他们提供的函数
      config.onMouseoutSquare(square, piece, deepCopy(currentPosition), currentOrientation)
    }

    // -------------------------------------------------------------------------
    // 初始化
    // -------------------------------------------------------------------------

    // 添加事件监听
    function addEvents () {
      // 阻止“图像拖动”
      $('body').on('mousedown mousemove', '.' + CSS.piece, stopDefault)

      // 鼠标拖动棋子
      $board.on('mousedown', '.' + CSS.square, mousedownSquare)
      $container.on('mousedown', '.' + CSS.sparePieces + ' .' + CSS.piece, mousedownSparePiece)

      // 鼠标进入/离开方块
      $board
        .on('mouseenter', '.' + CSS.square, mouseenterSquare)
        .on('mouseleave', '.' + CSS.square, mouseleaveSquare)

      // 棋子拖动
      var $window = $(window)
      $window
        .on('mousemove', throttledMousemoveWindow)
        .on('mouseup', mouseupWindow)

      // 触摸拖动棋子
      if (isTouchDevice()) {
        $board.on('touchstart', '.' + CSS.square, touchstartSquare)
        $container.on('touchstart', '.' + CSS.sparePieces + ' .' + CSS.piece, touchstartSparePiece)
        $window
          .on('touchmove', throttledTouchmoveWindow)
          .on('touchend', touchendWindow)
      }
    }
    // 初始化 DOM 元素
    function initDOM () {
      // 为将要创建的所有元素创建唯一的 ID
      createElIds()

      // 构建棋盘并将其保存在内存中
      $container.html(buildContainerHTML(config.sparePieces))
      $board = $container.find('.' + CSS.board)

      if (config.sparePieces) {
        $sparePiecesTop = $container.find('.' + CSS.sparePiecesTop)
        $sparePiecesBottom = $container.find('.' + CSS.sparePiecesBottom)
      }

      // 创建拖动棋子
      var draggedPieceId = uuid()
      $('body').append(buildPieceHTML('wP', true, draggedPieceId))
      $draggedPiece = $('#' + draggedPieceId)

      // TODO: 如果棋盘不再在 DOM 中，需要移除这个拖动的棋子元素

      // 获取边框大小
      boardBorderSize = parseInt($board.css('borderLeftWidth'), 10)

      // 设置大小并绘制棋盘
      widget.resize()
    }

    // -------------------------------------------------------------------------
    // 初始化
    // -------------------------------------------------------------------------

    setInitialState()
    initDOM()
    addEvents()

    // 返回 widget 对象
    return widget
  } // end constructor

  // TODO: 在这里执行模块导出

  // 将 Chessboard 对象赋给 window['Chessboard']
  window['Chessboard'] = constructor

  // 支持旧版 ChessBoard 名称
  window['ChessBoard'] = window['Chessboard']

  // 暴露实用函数
  window['Chessboard']['fenToObj'] = fenToObj
  window['Chessboard']['objToFen'] = objToFen
# 结束匿名包装函数
})()
```