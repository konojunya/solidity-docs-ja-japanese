.. index:: !mapping
.. _mapping-types:

マッピング型
=============

.. Mapping types use the syntax ``mapping(_KeyType => _ValueType)`` and variables
.. of mapping type are declared using the syntax ``mapping(_KeyType => _ValueType) _VariableName``.
.. The ``_KeyType`` can be any
.. built-in value type, ``bytes``, ``string``, or any contract or enum type. Other user-defined
.. or complex types, such as mappings, structs or array types are not allowed.
.. ``_ValueType`` can be any type, including mappings, arrays and structs.

マッピング型は ``mapping(_KeyType => _ValueType)`` の構文を使用し、マッピング型の変数は ``mapping(_KeyType => _ValueType) _VariableName`` の構文を使用して宣言されます。
``_KeyType`` には、任意の組み込み値型、 ``bytes`` 、 ``string`` 、または任意のコントラクト型や列挙型を使用できます。
マッピング、構造体、配列型など、その他のユーザー定義の型や複雑な型は使用できません。
``_ValueType`` は、マッピング、配列、構造体など、どのような型でも構いません。

.. You can think of mappings as `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_, which are virtually initialised
.. such that every possible key exists and is mapped to a value whose
.. byte-representation is all zeros, a type's :ref:`default value <default-value>`.
.. The similarity ends there, the key data is not stored in a
.. mapping, only its ``keccak256`` hash is used to look up the value.

マッピングは `ハッシュテーブル <https://en.wikipedia.org/wiki/Hash_table>`_ と考えることができ、ありとあらゆるキーが存在するように仮想的に初期化され、バイト表現がすべてゼロである値（型の :ref:`デフォルト値<default-value>` ）にマッピングされています。
キーデータはマッピングには保存されず、 ``keccak256`` ハッシュのみが値の検索に使用されるという点で似ています。

.. Because of this, mappings do not have a length or a concept of a key or
.. value being set, and therefore cannot be erased without extra information
.. regarding the assigned keys (see :ref:`clearing-mappings`).

このため、マッピングには長さや、キーや値が設定されているという概念がなく、割り当てられたキーに関する余分な情報（ :ref:`clearing-mappings` 参照）がないと消すことができません。

.. Mappings can only have a data location of ``storage`` and thus
.. are allowed for state variables, as storage reference types
.. in functions, or as parameters for library functions.
.. They cannot be used as parameters or return parameters
.. of contract functions that are publicly visible.
.. These restrictions are also true for arrays and structs that contain mappings.

マッピングのデータロケーションは ``storage`` のみであるため、状態変数、関数内のストレージ参照型、ライブラリ関数のパラメータとして使用できます。
これらは、一般に公開されているコントラクト関数のパラメータやリターンパラメータとしては使用できません。
これらの制限は、マッピングを含む配列や構造体にも当てはまります。

.. You can mark state variables of mapping type as ``public`` and Solidity creates a
.. :ref:`getter <visibility-and-getters>` for you. The ``_KeyType`` becomes a parameter for the getter.
.. If ``_ValueType`` is a value type or a struct, the getter returns ``_ValueType``.
.. If ``_ValueType`` is an array or a mapping, the getter has one parameter for
.. each ``_KeyType``, recursively.

マッピング型の状態変数を ``public`` としてマークすると、Solidityが :ref:`ゲッター <visibility-and-getters>` を作成してくれます。 ``_KeyType`` はゲッターのパラメータになります。
``_ValueType`` が値型または構造体の場合、ゲッターは ``_ValueType`` を返します。
``_ValueType`` が配列やマッピングの場合は、ゲッターは ``_KeyType`` ごとに1つのパラメータを再帰的に持ちます。

.. In the example below, the ``MappingExample`` contract defines a public ``balances``
.. mapping, with the key type an ``address``, and a value type a ``uint``, mapping
.. an Ethereum address to an unsigned integer value. As ``uint`` is a value type, the getter
.. returns a value that matches the type, which you can see in the ``MappingUser``
.. contract that returns the value at the specified address.

以下の例では、 ``MappingExample`` コントラクトがパブリック ``balances`` マッピングを定義しており、キー型は ``address`` 、値型は ``uint`` で、Ethereumアドレスを符号なし整数値にマッピングしています。
``uint`` は値型なので、ゲッターは型にマッチした値を返しますが、 ``MappingUser`` コントラクトでは指定されたアドレスの値を返しているのがわかります。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;

    contract MappingExample {
        mapping(address => uint) public balances;

        function update(uint newBalance) public {
            balances[msg.sender] = newBalance;
        }
    }

    contract MappingUser {
        function f() public returns (uint) {
            MappingExample m = new MappingExample();
            m.update(100);
            return m.balances(address(this));
        }
    }

.. The example below is a simplified version of an
.. `ERC20 token <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol>`_.
.. ``_allowances`` is an example of a mapping type inside another mapping type.
.. The example below uses ``_allowances`` to record the amount someone else is allowed to withdraw from your account.

下の例は、 `ERC20トークン <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol>`_ を簡略化したものです。
``_allowances`` は、別のマッピング型の中にマッピング型がある例です。
以下の例では、 ``_allowances`` を使って、他の人があなたのアカウントから引き出すことができる金額を記録しています。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.22 <0.9.0;

    contract MappingExample {

        mapping (address => uint256) private _balances;
        mapping (address => mapping (address => uint256)) private _allowances;

        event Transfer(address indexed from, address indexed to, uint256 value);
        event Approval(address indexed owner, address indexed spender, uint256 value);

        function allowance(address owner, address spender) public view returns (uint256) {
            return _allowances[owner][spender];
        }

        function transferFrom(address sender, address recipient, uint256 amount) public returns (bool) {
            _transfer(sender, recipient, amount);
            approve(sender, msg.sender, amount);
            return true;
        }

        function approve(address owner, address spender, uint256 amount) public returns (bool) {
            require(owner != address(0), "ERC20: approve from the zero address");
            require(spender != address(0), "ERC20: approve to the zero address");

            _allowances[owner][spender] = amount;
            emit Approval(owner, spender, amount);
            return true;
        }

        function _transfer(address sender, address recipient, uint256 amount) internal {
            require(sender != address(0), "ERC20: transfer from the zero address");
            require(recipient != address(0), "ERC20: transfer to the zero address");

            _balances[sender] -= amount;
            _balances[recipient] += amount;
            emit Transfer(sender, recipient, amount);
        }
    }

.. index:: !iterable mappings
.. _iterable-mappings:

イテレート可能なマッピング
------------------------------

.. You cannot iterate over mappings, i.e. you cannot enumerate their keys.
.. It is possible, though, to implement a data structure on
.. top of them and iterate over that. For example, the code below implements an
.. ``IterableMapping`` library that the ``User`` contract then adds data too, and
.. the ``sum`` function iterates over to sum all the values.

マッピングはイテレートできません。つまり、キーを列挙することもできません。
しかし、マッピングの上にデータ構造を実装し、その上で反復処理を行うことは可能です。
例えば、以下のコードでは、 ``IterableMapping`` ライブラリを実装し、 ``User`` コントラクトがデータを追加し、 ``sum`` 関数がすべての値を合計するために反復処理を行います。

.. code-block:: solidity
    :force:

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.8 <0.9.0;

    struct IndexValue { uint keyIndex; uint value; }
    struct KeyFlag { uint key; bool deleted; }

    struct itmap {
        mapping(uint => IndexValue) data;
        KeyFlag[] keys;
        uint size;
    }

    library IterableMapping {
        function insert(itmap storage self, uint key, uint value) internal returns (bool replaced) {
            uint keyIndex = self.data[key].keyIndex;
            self.data[key].value = value;
            if (keyIndex > 0)
                return true;
            else {
                keyIndex = self.keys.length;
                self.keys.push();
                self.data[key].keyIndex = keyIndex + 1;
                self.keys[keyIndex].key = key;
                self.size++;
                return false;
            }
        }

        function remove(itmap storage self, uint key) internal returns (bool success) {
            uint keyIndex = self.data[key].keyIndex;
            if (keyIndex == 0)
                return false;
            delete self.data[key];
            self.keys[keyIndex - 1].deleted = true;
            self.size --;
        }

        function contains(itmap storage self, uint key) internal view returns (bool) {
            return self.data[key].keyIndex > 0;
        }

        function iterate_start(itmap storage self) internal view returns (uint keyIndex) {
            return iterate_next(self, type(uint).max);
        }

        function iterate_valid(itmap storage self, uint keyIndex) internal view returns (bool) {
            return keyIndex < self.keys.length;
        }

        function iterate_next(itmap storage self, uint keyIndex) internal view returns (uint r_keyIndex) {
            keyIndex++;
            while (keyIndex < self.keys.length && self.keys[keyIndex].deleted)
                keyIndex++;
            return keyIndex;
        }

        function iterate_get(itmap storage self, uint keyIndex) internal view returns (uint key, uint value) {
            key = self.keys[keyIndex].key;
            value = self.data[key].value;
        }
    }

    // 使用方法
    contract User {
        // データを保持する構造体
        itmap data;
        // データ型にライブラリ関数を適用する。
        using IterableMapping for itmap;

        // 何かを挿入する
        function insert(uint k, uint v) public returns (uint size) {
            // これは IterableMapping.insert(data, k, v) を呼び出します。
            data.insert(k, v);
            // 構造体のメンバーにアクセスすることは可能ですが、構造体をいじらないように注意する必要があります。
            return data.size;
        }

        // 保存されているすべてのデータの合計を計算する。
        function sum() public view returns (uint s) {
            for (
                uint i = data.iterate_start();
                data.iterate_valid(i);
                i = data.iterate_next(i)
            ) {
                (, uint value) = data.iterate_get(i);
                s += value;
            }
        }
    }

