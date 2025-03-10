.. index:: ! type;reference, ! reference type, storage, memory, location, array, struct

.. _reference-types:

参照型
===============

参照型の値は、複数の異なる名前で変更できます。
値型の変数が使われるたびに独立したコピーを得ることができる値型とは対照的です。
そのため、参照型は値型よりも慎重に扱う必要があります。
現在、参照型は構造体、配列、マッピングで構成されています。
参照型を使用する場合は、その型が格納されているデータ領域を常に明示的に提供する必要があります。
``memory`` （ライフタイムが外部の関数呼び出しに限定される）、 ``storage`` （状態変数が格納されている場所で、ライフタイムがコントラクトのライフタイムに限定される）、 ``calldata`` （関数の引数が格納されている特別なデータの場所）のいずれかになります。

データロケーションを変更する割り当てや型変換は、常に自動コピー操作が発生しますが、同じデータロケーション内の割り当ては、ストレージ型の場合、一部のケースでしかコピーされません。

.. _data-location:

データロケーション
--------------------

すべての参照型には、それがどこに保存されているかについて、「データロケーション」という追加のアノテーションがあります。
データロケーションは3つあります。 ``memory`` 、 ``storage`` 、 ``calldata`` です。
コールデータは、関数の引数が格納される、変更不可能で永続性のない領域で、ほとんどメモリのように動作します。

.. note::

    できれば、データの場所として ``calldata`` を使うようにしましょう。
    コピーを避けることができますし、データが変更されないようにすることもできます。
    ``calldata`` 型のデータを持つ配列や構造体は、関数から返すことができますが、そのような型を代入することはできません。

.. note::

    バージョン0.6.9以前では、参照型引数のデータロケーションは、外部関数では ``calldata`` 、パブリック関数では ``memory`` 、内部およびプライベート関数では ``memory`` または ``storage`` に制限されていました。
    現在では、 ``memory`` と ``calldata`` は、その可視性に関わらず、すべての関数で許可されています。

.. note::

    バージョン0.5.0までは、データロケーションを省略でき、変数の種類や関数の種類などに応じて異なるロケーションをデフォルトとしていましたが、現在ではすべての複合型でデータのロケーションを明示的に指定しなければなりません。

.. _data-location-assignment:

データロケーションと代入の挙動
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

データロケーションは、データの永続性だけでなく、代入のセマンティクスにも関係します。

* ``storage`` と ``memory`` の間（または ``calldata`` から）の代入では、常に独立したコピーが作成されます。

* ``memory`` から ``memory`` への代入では、参照のみが作成されます。つまり、あるメモリ変数の変更は、同じデータを参照している他のすべてのメモリ変数にも反映されるということです。

* ``storage`` から **ローカル** ストレージ変数への代入もまた、参照のみを代入します。

* その他の ``storage`` への代入は、常にコピーされます。このケースの例としては、状態変数やストレージ構造体型のローカル変数のメンバへの代入がありますが、ローカル変数自体が単なる参照であっても同様です。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.5.0 <0.9.0;

    contract C {
        // xのデータロケーションはstorageです。
        // データロケーションを省略できるのはここだけです。
        uint[] x;

        // memoryArray のデータロケーションはメモリです。
        function f(uint[] memory memoryArray) public {
            x = memoryArray; // 配列全体をストレージにコピーします。
            uint[] storage y = x; // ポインタが代入され、yのデータロケーションはストレージになります。
            y[7]; // 8番目の要素を返します。
            y.pop(); // yを通してxも修正されます。
            delete x; // 配列がクリアされ、yも修正されます。
            // ストレージに新しい一時的な無名配列を作成する必要がありますが、ストレージは「静的」に割り当てられているため、次のようにするとうまくいきません。
            // y = memoryArray;
            // ポインターはリセットされますが、ポインターの指し示す場所がないため、この方法もうまくいきません。
            // delete y;
            g(x); // gを呼び出し、xへの参照を渡します。
            h(x); // hを呼び出し、独立した一時的なコピーをメモリ上に作成します。
        }

        function g(uint[] storage) internal pure {}
        function h(uint[] memory) public pure {}
    }

.. index:: ! array

.. _arrays:

配列
------

配列は、コンパイル時に固定されたサイズも動的なサイズも持つことができます。

固定サイズ ``k`` 、要素型 ``T`` の配列の型は ``T[k]`` 、動的サイズの配列は ``T[]`` と書きます。

例えば、 ``uint`` の動的配列を5個並べた配列は ``uint[][5]`` と書きます。この表記法は、他のいくつかの言語とは逆になっています。Solidityでは、たとえ ``X`` がそれ自体配列であっても、 ``X[3]`` は常に ``X`` 型の3つの要素を含む配列です。これは、Cなどの他の言語ではそうではありません。

インデックスはゼロベースで、アクセスは宣言とは逆方向になります。

例えば、変数 ``uint[][5] memory x`` がある場合、3番目の動的配列の7番目の ``uint`` にアクセスするには ``x[2][6]`` を使い、3番目の動的配列にアクセスするには ``x[2]`` を使います。繰り返しになりますが、配列にもなる ``T`` 型に対して配列 ``T[5] a`` がある場合、 ``a[2]`` は常に ``T`` 型です。

配列の要素は、マッピングや構造体など、どのような型でもよいです。一般的な型の制限が適用され、マッピングは ``storage`` データの場所にしか保存できず、一般に公開されている関数には :ref:`ABI型<ABI>` のパラメータが必要となります。

状態変数の配列に ``public`` をマークして、Solidityに :ref:`getter <visibility-and-getters>` を作成させることが可能です。数値インデックスは、getterの必須パラメータとなります。

配列の終端を超えてアクセスすると、アサーションが失敗します。メソッド ``.push()`` と ``.push(value)`` は、配列の最後に新しい要素を追加するために使用でき、 ``.push()`` はゼロ初期化された要素を追加し、その要素への参照を返します。

.. index:: ! string, ! bytes

.. _strings:

.. _bytes:

配列としての ``bytes`` と ``string``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``bytes`` 型と ``string`` 型の変数は、特殊な配列です。
``bytes`` 型は ``bytes1[]`` と似ていますが、calldataとメモリにしっかりと詰め込まれています。
``string`` は ``bytes`` と同じですが、長さやインデックスのアクセスはできません。

Solidityには文字列操作関数はありませんが、サードパーティ製の文字列ライブラリがあります。
また、 ``keccak256(abi.encodePacked(s1)) == keccak256(abi.encodePacked(s2))`` を使って2つの文字列をそのkeccak256-hashで比較したり、 ``bytes.concat(bytes(s1), bytes(s2))`` を使って2つの文字列を連結できます。

``bytes1[]`` は要素間に31個のパディングバイトを追加するので、 ``bytes1[]`` よりも ``bytes`` を使用した方が安価です。
原則として、任意の長さの生バイトデータには ``bytes`` を、任意の長さの文字列（UTF-8）データには ``string`` を使用してください。
長さを一定のバイト数に制限できる場合は、値型 ``bytes1`` 〜 ``bytes32`` のいずれかを必ず使用してください。その方がはるかに安価です）。

.. note::

    ``s`` という文字列のバイト表現にアクセスしたい場合は、 ``bytes(s).length`` / ``bytes(s)[7] = 'x';`` を使います。
    UTF-8表現の低レベルバイトにアクセスしているのであって、個々の文字にアクセスしているわけではないことに注意してください。

.. index:: ! bytes-concat

.. _bytes-concat:

``bytes.concat`` 関数
^^^^^^^^^^^^^^^^^^^^^^^^^

``bytes.concat`` を使って可変数の ``bytes`` や ``bytes1 ... bytes32`` を連結できます。
この関数は、パディングされていない引数の内容を含む単一の ``bytes memory`` 配列を返します。
文字列のパラメータや他の型を使いたい場合は、まず ``bytes`` や ``bytes1`` / ... / ``bytes32`` に変換する必要があります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.4;

    contract C {
        bytes s = "Storage";
        function f(bytes calldata c, string memory m, bytes16 b) public view {
            bytes memory a = bytes.concat(s, c, c[:2], "Literal", bytes(m), b);
            assert((s.length + c.length + 2 + 7 + bytes(m).length + 16) == a.length);
        }
    }

引数なしで ``bytes.concat`` を呼び出すと、空の ``bytes`` 配列が返されます。

.. index:: ! array;allocating, new

メモリ配列のアロケート
^^^^^^^^^^^^^^^^^^^^^^^^

動的な長さを持つメモリ配列は、 ``new`` 演算子を使って作成できます。
ストレージ配列とは対照的に、メモリ配列のサイズを変更できません（例えば、 ``.push`` メンバ関数は使用できません）。
必要なサイズを事前に計算するか、新しいメモリ配列を作成してすべての要素をコピーする必要があります。

Solidityのすべての変数と同様に、新しく割り当てられた配列の要素は、常に :ref:`デフォルト値<default-value>` で初期化されます。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    contract C {
        function f(uint len) public pure {
            uint[] memory a = new uint[](7);
            bytes memory b = new bytes(len);
            assert(a.length == 7);
            assert(b.length == len);
            a[6] = 8;
        }
    }

.. index:: ! array;literals, ! inline;arrays

配列リテラル
^^^^^^^^^^^^^^

配列リテラルは、1つまたは複数の式を角括弧（ ``[...]`` ）で囲んだコンマ区切りのリストです。
例えば、 ``[1, a, f(3)]`` です。配列リテラルの型は以下のように決定されます。

これは、常に静的サイズのメモリ配列で、その長さは式の数です。

配列の基本型は、リストの最初の式の型で、他のすべての式が暗黙的に変換できるようになっています。
これができない場合は型エラーとなります。

すべての要素に変換できる型があるだけでは不十分です。要素の一つがその型でなければなりません。

下の例では、それぞれの定数の型が ``uint8`` であることから、 ``[1, 2, 3]`` の型は ``uint8[3] memory`` となります。結果を ``uint[3] memory`` 型にしたい場合は、最初の要素を ``uint`` に変換する必要があります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    contract C {
        function f() public pure {
            g([uint(1), 2, 3]);
        }
        function g(uint[3] memory) public pure {
            // ...
        }
    }

配列リテラル ``[1, -1]`` が無効なのは、最初の式の型が ``uint8`` であるのに対し、2番目の式の型が ``int8`` であり、両者を暗黙的に変換できないからです。これを動作させるには、例えば ``[int8(1), -1]`` を使用します。

異なる型の固定サイズのメモリ配列は、（基底型が変換できても）相互に変換できないため、二次元配列リテラルを使用する場合は、常に共通の基底型を明示的に指定する必要があります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    contract C {
        function f() public pure returns (uint24[2][4] memory) {
            uint24[2][4] memory x = [[uint24(0x1), 1], [0xffffff, 2], [uint24(0xff), 3], [uint24(0xffff), 4]];
            // 以下のようにすると、内側の配列の一部が正しい型でないため、うまくいきません。
            // uint[2][4] memory x = [[0x1, 1], [0xffffff, 2], [0xff, 3], [0xffff, 4]];
            return x;
        }
    }

固定サイズのメモリ配列を、動的サイズのメモリ配列に代入することはできません。つまり、以下のことはできません。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;

    // これではコンパイルできません。
    contract C {
        function f() public {
            // 次の行は、uint[3]メモリをuint[]メモリに変換できないため、型エラーが発生します。
            uint[] memory x = [uint(1), 3, 4];
        }
    }

将来的にはこの制限を解除する予定ですが、ABIでの配列の渡し方の関係で複雑になっています。

動的なサイズの配列を初期化したい場合は、個々の要素を代入する必要があります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    contract C {
        function f() public pure {
            uint[] memory x = new uint[](3);
            x[0] = 1;
            x[1] = 3;
            x[2] = 4;
        }
    }

.. index:: ! array;length, length, push, pop, !array;push, !array;pop

.. _array-members:

配列のメンバー
^^^^^^^^^^^^^^^^

**length**:
    配列は、要素数を表す ``length`` メンバを持ちます。
    メモリー配列の長さは、作成時に固定されます（ただし、動的、つまり実行時のパラメータに依存することがあります）。
**push()**:
    動的ストレージ配列と ``bytes`` （ ``string`` ではありません）には、 ``push()`` というメンバ関数があり、配列の最後にゼロ初期化された要素を追加するのに使用できます。
    この関数は、要素への参照を返すので、 ``x.push().t = 2`` や ``x.push() = b`` のように使用できます。
**push(x)**:
    動的ストレージ配列と ``bytes``（ ``string`` ではありません）には、 ``push(x)`` というメンバ関数があり、配列の最後に与えられた要素を追加するのに使用できます。
    この関数は何も返しません。 
**pop**:
    動的ストレージ配列と ``bytes`` （ ``string`` ではありません）には ``pop`` というメンバ関数があり、配列の最後から要素を削除するのに使用できます。
    この関数は、削除された要素に対して :ref:`delete<delete>`  を暗黙的に呼び出します。

.. note::

    ``push()`` を呼び出してストレージ配列の長さを増加させると、ストレージがゼロ初期化されるため、ガスコストが一定になります。
    一方、 ``pop()`` を呼び出して長さを減少させると、削除される要素の「サイズ」に依存するコストが発生します。
    その要素が配列の場合は、 :ref:`delete<delete>` を呼び出すのと同様に、削除された要素を明示的にクリアすることが含まれるため、非常にコストがかかります。

.. note::

    配列の配列を（publicではなく）外部関数で使用するには、ABI coder v2を有効にする必要があります。

.. note::

    Byzantium以前のEVMバージョンでは、関数呼び出しから返される動的配列にアクセスできませんでした。
    動的配列を返す関数を呼び出す場合は、必ずByzantiumモードに設定されたEVMを使用してください。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.0 <0.9.0;

    contract ArrayContract {
        uint[2**20] m_aLotOfIntegers;
        // 以下は動的配列のペアではなく、ペアの動的配列（つまり長さ2の固定サイズ配列のペア）であることに注意してください。
        // T[]はT自体が配列であっても、常にTの動的配列となります。
        // すべての状態変数のデータロケーションはストレージです。
        bool[2][] m_pairsOfFlags;

        // newPairsはメモリに格納されます - パブリックコントラクト関数の引数として唯一の選択肢です。
        function setAllFlagPairs(bool[2][] memory newPairs) public {
            // ストレージ配列への代入は、 ``newPairs`` のコピーを実行し、完全な配列 ``m_pairsOfFlags`` を置き換えます。
            m_pairsOfFlags = newPairs;
        }

        struct StructType {
            uint[] contents;
            uint moreInfo;
        }
        StructType s;

        function f(uint[] memory c) public {
            // ``s`` への参照を ``g`` に格納します。
            StructType storage g = s;
            // ``s.moreInfo`` も変更します。
            g.moreInfo = 2;
            // コピーを代入します。
            // なぜなら ``g.contents`` はローカル変数ではなく、ローカル変数のメンバだからです。
            g.contents = c;
        }

        function setFlagPair(uint index, bool flagA, bool flagB) public {
            // 存在しないインデックスにアクセスすると、例外が発生します。
            m_pairsOfFlags[index][0] = flagA;
            m_pairsOfFlags[index][1] = flagB;
        }

        function changeFlagArraySize(uint newSize) public {
            // 配列の長さを変更するには、push と pop を使用するのが唯一の方法です。
            if (newSize < m_pairsOfFlags.length) {
                while (m_pairsOfFlags.length > newSize)
                    m_pairsOfFlags.pop();
            } else if (newSize > m_pairsOfFlags.length) {
                while (m_pairsOfFlags.length < newSize)
                    m_pairsOfFlags.push();
            }
        }

        function clear() public {
            // これらは、配列を完全にクリアします。
            delete m_pairsOfFlags;
            delete m_aLotOfIntegers;
            // これも同じ効果です。
            m_pairsOfFlags = new bool[2][](0);
        }

        bytes m_byteData;

        function byteArrays(bytes memory data) public {
            // バイト配列（"bytes"）はパディングなしで格納されるため異なりますが、"uint8[]"と同じように扱うことができます。
            m_byteData = data;
            for (uint i = 0; i < 7; i++)
                m_byteData.push();
            m_byteData[3] = 0x08;
            delete m_byteData[2];
        }

        function addFlag(bool[2] memory flag) public returns (uint) {
            m_pairsOfFlags.push(flag);
            return m_pairsOfFlags.length;
        }

        function createMemoryArray(uint size) public pure returns (bytes memory) {
            // 動的メモリ配列は `new` を用いて作成する。
            uint[2][] memory arrayOfPairs = new uint[2][](size);

            // インライン配列は常に静的サイズであり、リテラルのみを使用する場合は、少なくとも1つの型を提供する必要があります。
            arrayOfPairs[0] = [uint(1), 2];

            // ダイナミックバイト列を作成する。
            bytes memory b = new bytes(200);
            for (uint i = 0; i < b.length; i++)
                b[i] = bytes1(uint8(i));
            return b;
        }
    }

.. index:: ! array;slice

.. _array-slices:

配列のスライス
---------------

配列のスライスは、配列の連続した部分のビューです。
スライスは ``x[start:end]`` と書き、 ``start`` と ``end`` はuint256型になる（または暗黙のうちに変換できる）式です。
スライスの最初の要素は ``x[start]`` で、最後の要素は ``x[end - 1]`` です。

``start`` が ``end`` より大きい場合や、 ``end`` が配列の長さより大きい場合は、例外が発生します。

``start`` と ``end`` はどちらもオプションです。 ``start`` はデフォルトで ``0`` 、 ``end`` はデフォルトで配列の長さになります。

配列スライスは、メンバーを持ちません。
スライスは、基礎となる型の配列に暗黙的に変換可能で、インデックスアクセスをサポートします。
インデックスアクセスは、基礎となる配列での絶対的なものではなく、スライスの開始点からの相対的なものです。

配列スライスには型名がありません。
つまり、どの変数も配列スライスを型として持つことはできず、中間式にのみ存在することになります。

.. note::

    現在、配列スライスはcalldata配列に対してのみ実装されています。

配列スライスは、関数のパラメータで渡された二次データをABIデコードするのに便利です。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.5 <0.9.0;
    contract Proxy {
        /// @dev プロキシ（すなわちこのコントラクト）で管理するクライアントコントラクトのアドレス
        address client;

        constructor(address _client) {
            client = _client;
        }

        /// 引数のアドレスの基本的な検証を行った後、クライアントが実装する"setOwner(address)"のフォワードコール
        function forward(bytes calldata _payload) external {
            bytes4 sig = bytes4(_payload[:4]);
            // 切り捨て処理のため、bytes4(_payload)も同じ処理
            // bytes4 sig = bytes4(_payload);
            if (sig == bytes4(keccak256("setOwner(address)"))) {
                address owner = abi.decode(_payload[4:], (address));
                require(owner != address(0), "Address of owner cannot be zero.");
            }
            (bool status,) = client.delegatecall(_payload);
            require(status, "Forwarded call failed.");
        }
    }

.. index:: ! struct, ! type;struct

.. _structs:

構造体
-------

Solidityでは、構造体の形で新しい型を定義する方法を提供しており、次の例のようになります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.0 <0.9.0;

    // 2つのフィールドを持つ新しい型を定義します。
    // 構造体をコントラクトの外部で宣言すると、複数のコントラクトで共有できるようになります。
    // ここでは、これはあまり必要ありません。
    struct Funder {
        address addr;
        uint amount;
    }

    contract CrowdFunding {
        // 構造体はコントラクトの内部で定義することもでき、その場合、その内部および派生コントラクトでのみ認識できるようになります。
        struct Campaign {
            address payable beneficiary;
            uint fundingGoal;
            uint numFunders;
            uint amount;
            mapping (uint => Funder) funders;
        }

        uint numCampaigns;
        mapping (uint => Campaign) campaigns;

        function newCampaign(address payable beneficiary, uint goal) public returns (uint campaignID) {
            campaignID = numCampaigns++; // campaignIDは返り値です。 
            // "campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0)"は、
            // 右側がマッピングを含むメモリ構造体"Campaign"を作成するため、使用することはできません。
            Campaign storage c = campaigns[campaignID];
            c.beneficiary = beneficiary;
            c.fundingGoal = goal;
        }

        function contribute(uint campaignID) public payable {
            Campaign storage c = campaigns[campaignID];
            // 指定された値で初期化された新しい一時的なメモリ構造体を作成し、ストレージにコピーします。
            // Funder(msg.sender, msg.value) を使用して初期化することもできます。
            c.funders[c.numFunders++] = Funder({addr: msg.sender, amount: msg.value});
            c.amount += msg.value;
        }

        function checkGoalReached(uint campaignID) public returns (bool reached) {
            Campaign storage c = campaigns[campaignID];
            if (c.amount < c.fundingGoal)
                return false;
            uint amount = c.amount;
            c.amount = 0;
            c.beneficiary.transfer(amount);
            return true;
        }
    }

このコントラクトは、クラウドファンディングのコントラクトの機能をすべて提供するものではありませんが、構造体を理解するために必要な基本的な概念が含まれています。
構造体はマッピングや配列の内部で使用でき、構造体自身もマッピングや配列を含むことができます。

構造体に自身の型のメンバーを含めることはできませんが、構造体自体をマッピングメンバーの値の型にしたり、構造体にその型の動的サイズの配列を含めることはできます。
構造体のサイズは有限である必要があるため、この制限は必要です。

すべての関数で、構造体型がデータロケーション ``storage`` のローカル変数に代入されていることに注目してください。
これは構造体をコピーするのではなく、参照を保存するだけなので、ローカル変数のメンバーへの代入は実際にステートに書き込まれます。

もちろん、 ``campaigns[campaignID].amount = 0`` のようにローカル変数に代入せずに、構造体のメンバーに直接アクセスすることもできます。

.. note::

    Solidity 0.7.0までは、ストレージのみの型（マッピングなど）のメンバーを含むメモリ構造が許可されており、上の例の ``campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0)`` のような代入が機能し、それらのメンバーを静かにスキップしていました。
