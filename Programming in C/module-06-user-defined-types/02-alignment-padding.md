# Alignment and Padding

Implementations may insert padding between structure members and at the end to
satisfy alignment requirements. Consequently, `sizeof(struct)` is not
necessarily the sum of its member sizes. Member order can change the padding;
reordering fields may reduce the size on a given ABI.

There is no portable formula for every structure's exact size without
implementation-specific alignment rules. Use the ABI/compiler assumptions
supplied by a question.
