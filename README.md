# magnet_link_homework

```python
import itertools as it
import hashlib
import string
import os
import re


# https://github.com/utdemir/bencoder/
def encode(obj):
    """
    bencodes given object. Given object should be a int,
    bytes, list or dict. If a str is given, it'll be
    encoded as UTF-8.

    >>> [encode(i) for i in (-2, 42, b"answer", b"")] \
            == [b'i-2e', b'i42e', b'6:answer', b'0:']
    True
    >>> encode([b'a', 42, [13, 14]]) == b'l1:ai42eli13ei14eee'
    True
    >>> encode({b'bar': b'spam', b'foo': 42, b'mess': [1, b'c']}) \
            == b'd3:bar4:spam3:fooi42e4:messli1e1:cee'
    True
    """

    if isinstance(obj, int):
        return b"i" + str(obj).encode() + b"e"
    elif isinstance(obj, bytes):
        return str(len(obj)).encode() + b":" + obj
    elif isinstance(obj, str):
        return encode(obj.encode("utf-8"))
    elif isinstance(obj, list):
        return b"l" + b"".join(map(encode, obj)) + b"e"
    elif isinstance(obj, dict):
        if all(isinstance(i, bytes) for i in obj.keys()):
            items = list(obj.items())
            items.sort()
            return b"d" + b"".join(map(encode, it.chain(*items))) + b"e"
        else:
            raise ValueError("dict keys should be bytes " + str(obj.keys()))
    raise ValueError("Allowed types: int, bytes, list, dict; not %s", type(obj))


def decode(s):
    """
    Decodes given bencoded bytes object.

    >>> decode(b'i-42e')
    -42
    >>> decode(b'4:utku') == b'utku'
    True
    >>> decode(b'li1eli2eli3eeee')
    [1, [2, [3]]]
    >>> decode(b'd3:bar4:spam3:fooi42ee') == {b'bar': b'spam', b'foo': 42}
    True
    """
    def decode_first(s):
        if s.startswith(b"i"):
            match = re.match(b"i(-?\\d+)e", s)
            return int(match.group(1)), s[match.span()[1]:]
        elif s.startswith(b"l") or s.startswith(b"d"):
            l = []
            rest = s[1:]
            while not rest.startswith(b"e"):
                elem, rest = decode_first(rest)
                l.append(elem)
            rest = rest[1:]
            if s.startswith(b"l"):
                return l, rest
            else:
                return {i: j for i, j in zip(l[::2], l[1::2])}, rest
        elif any(s.startswith(i.encode()) for i in string.digits):
            m = re.match(b"(\\d+):", s)
            length = int(m.group(1))
            rest_i = m.span()[1]
            start = rest_i
            end = rest_i + length
            return s[start:end], s[end:]
        else:
            raise ValueError("Malformed input.")

    if isinstance(s, str):
        s = s.encode("utf-8")

    ret, rest = decode_first(s)
    if rest:
        raise ValueError("Malformed input.")
    return ret


# Минимальный размер блока данных
DATA_BLOCK_SIZE = 16 * 1024
PIECE_LENGTH = DATA_BLOCK_SIZE * 4


def join_hash_stack(a_hash_stack):
    # Если 2 верхних элемента стэка имеют одинаковые уровни
    while len(a_hash_stack) > 1 and a_hash_stack[-1][0] == a_hash_stack[-2][0]:
        fst = a_hash_stack.pop()
        snd = a_hash_stack.pop()
        # Складываем уровни и хэши
        joined = (fst[0] + snd[0], hashlib.sha256(snd[1] + fst[1]).digest())
        a_hash_stack.append(joined)


def hash_file(a_filepath):
    with open(a_filepath, 'rb') as file:
        hash_stack = []
        data = file.read(DATA_BLOCK_SIZE)
        while data:
            # 1 -> слой первого уровня
            hash_stack.append((1, hashlib.sha256(data).digest()))
            join_hash_stack(hash_stack)

            data = file.read(DATA_BLOCK_SIZE)

    while len(hash_stack) > 1:
        # Добавляем еще блоки первого уровня с хэшем равным нулю, пока стэк не свернется
        # (http://www.bittorrent.org/beps/bep_0052.html pieces root)
        hash_stack.append((1, bytes(32)))
        join_hash_stack(hash_stack)

    return hash_stack[0][1]


def dir_as_infohash(a_path):
    if os.path.isfile(a_path):
        file_size = os.path.getsize(a_path)
        file_info = {
            b'': {
                b'length': file_size,
            }
        }
        if file_size > 0:
            file_info[b''][b'pieces root'] = hash_file(a_path)

        return file_info
    else:
        dirs = {}
        # Словарь должен быть отсортирован по ключам (http://www.bittorrent.org/beps/bep_0052.html bencoding)
        for entry in sorted([d.encode("utf8") for d in os.listdir(a_path)]):
            entry_path = os.path.join(a_path, entry.decode("utf8"))
            dirs[entry] = dir_as_infohash(entry_path)
        return dirs


def get_magnet_btmh(path: str) -> str:
    path = path.rstrip(os.sep)

    info_section = {
        b'name': os.path.basename(path),
        b'piece length': PIECE_LENGTH,
        b'file tree': None,
        b'meta version': 2,
    }

    if os.path.isfile(path):
        info_section[b'file tree'] = {os.path.basename(path).encode("utf8"): dir_as_infohash(path)}
    else:
        info_section[b'file tree'] = dir_as_infohash(path)

    return hashlib.sha256(encode(info_section)).hexdigest()


if __name__ == '__main__':
    btmh_dir = get_magnet_btmh("/home/dmitry/develop/nasm")
    print(f"Magnet link: magnet:?xt=urn:btmh:{btmh_dir}&dn=bittorrent-v2-test")

    btmh_file = get_magnet_btmh("/home/dmitry/develop/nasm/file.txt")
    print(f"Magnet link: magnet:?xt=urn:btmh:{btmh_file}&dn=bittorrent-v2-test")

```
