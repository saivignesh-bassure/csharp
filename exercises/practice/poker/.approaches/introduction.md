# Introduction

TODO

## General guidance

TODO

## Approach: integer score

```csharp
public static class Poker
{
    public static IEnumerable<string> BestHands(IEnumerable<string> hands) =>
        hands.ToLookup(HandScore).MaxBy(g => g.Key);

    private static int HandScore(string hand) =>
        Math.Max(Parser.ParseHand(hand).Score, Parser.ParseHand(hand.Replace('A', '1')).Score);

    private record Card(int Rank, int Suit);

    private class Hand
    {
        private readonly int[] ranks;
        private readonly int[] rankCounts;
        private readonly int[] suitCounts;

        public Hand(Card[] cards)
        {
            ranks = cards.Select(card => card.Rank).OrderByDescending(rank => cards.Count(card => card.Rank == rank)).ThenByDescending(card => card).ToArray();
            suitCounts = cards.GroupBy(card => card.Suit).Select(grouping => grouping.Count()).OrderDescending().ToArray();
            rankCounts = cards.GroupBy(card => card.Rank).Select(grouping => grouping.Count()).OrderDescending().ToArray();
        }

        public int Score => ranks.Prepend(HandRank).Aggregate((total, value) => total * 14 + value);

        private int HandRank =>
            IsStraightFlush ? 9 :
            IsFourOfAKind ? 8 :
            IsFullHouse ? 7 :
            IsFlush ? 6 :
            IsStraight ? 5 :
            IsThreeOfAKind ? 4 :
            IsTwoPair ? 3 :
            IsOnePair ? 2 :
            1;

        private bool IsStraightFlush => IsFlush && IsStraight;
        private bool IsFlush => suitCounts is [5];
        private bool IsStraight => rankCounts is [1, 1, 1, 1, 1] && ranks[0] - ranks[4] == 4;
        private bool IsFourOfAKind => rankCounts is [4, 1];
        private bool IsFullHouse => rankCounts is [3, 2];
        private bool IsThreeOfAKind => rankCounts is [3, 1, 1];
        private bool IsTwoPair => rankCounts is [2, 2, 1];
        private bool IsOnePair => rankCounts is [2, 1, 1, 1];
    }

    private static class Parser
    {
        public static Hand ParseHand(string hand) => new(ParseCards(hand));
        private static Card[] ParseCards(string hand) => hand.Split(' ').Select(ParseCard).ToArray();
        private static Card ParseCard(string card) => new(Rank(card), Suit(card));
        private static int Rank(string card) => ".1234567890JQKA".IndexOf(card[^2]);
        private static int Suit(string card) => ".HSDC".IndexOf(card[^1]);
    }
}
```

This approach assign a single integer score to each hand, which makes finding the best hand(s) quite straightforward.
For more information, check the [integer score approach][approach-icomparer].

## Approach: `IComparer<T>`

```csharp
public static class Poker
{
    public static IEnumerable<string> BestHands(IEnumerable<string> hands)
    {
        var parsedHands = hands.ToDictionary(hand => hand, Parser.ParseHand);
        var bestHand = parsedHands.Values.Max();

        return parsedHands
            .Where(hand => hand.Value.CompareTo(bestHand) == 0)
            .Select(hand => hand.Key)
            .ToArray();
    }

    private enum Suit { Diamonds, Clubs, Hearts, Spades }

    private enum Rank { Two, Three, Four, Five, Six, Seven, Eight, Nine, Ten, Jack, Queen, King, Ace }

    private enum Category { HighCard, OnePair, TwoPair, ThreeOfAKind, Straight, Flush, FullHouse, FourOfAKind, StraightFlush }

    private record Card(Rank Rank, Suit Suit);

    private record Hand(Card[] Cards) : IComparable<Hand>
    {
        private readonly Rank[] ranks = Cards
            .Select(card => card.Rank)
            .OrderByDescending(rank => Cards.Count(card => card.Rank == rank))
            .ThenByDescending(card => card)
            .ToArray();

        private readonly int[] rankCounts = Cards
            .GroupBy(card => card.Rank)
            .Select(grouping => grouping.Count())
            .OrderDescending()
            .ToArray();

        public int CompareTo(Hand other) =>
            Category.CompareTo(other.Category) switch
            {
                < 0 => -1,
                > 0 => 1,
                0 => CategoryRanks.CompareTo(other.CategoryRanks)
            };

        private Category Category =>
            rankCounts switch
            {
                [1, 1, 1, 1, 1] when IsFlush && IsStraight => Category.StraightFlush,
                [4, 1] => Category.FourOfAKind,
                [3, 2] => Category.FullHouse,
                _ when IsFlush => Category.Flush,
                [1, 1, 1, 1, 1] when IsStraight => Category.Straight,
                [3, 1, 1] => Category.ThreeOfAKind,
                [2, 2, 1] => Category.TwoPair,
                [2, 1, 1, 1] => Category.OnePair,
                _ => Category.HighCard
            };

        private bool IsFlush => Cards.DistinctBy(card => card.Suit).Count() == 1;
        private bool IsStraight => IsRegularStraight || IsLowAceStraight;
        private bool IsRegularStraight => ranks[0] - ranks[4] is 4;
        private bool IsLowAceStraight => ranks[0] - ranks[1] is 9;

        private (Rank, Rank, Rank, Rank, Rank) CategoryRanks =>
            Category switch
            {
                Category.StraightFlush or Category.Straight when IsLowAceStraight => (ranks[1], ranks[2], ranks[3], ranks[4], ranks[0]),
                _ => (ranks[0], ranks[1], ranks[2], ranks[3], ranks[4])
            };
    }

    private static class Parser
    {
        public static Hand ParseHand(string hand) => new(ParseCards(hand));
        private static Card[] ParseCards(string hand) => hand.Split(' ').Select(ParseCard).ToArray();
        private static Card ParseCard(string card) => new(ParseRank(card), ParseSuit(card));

        private static Rank ParseRank(string card) =>
            card[0] switch
            {
                '2' => Rank.Two,
                '3' => Rank.Three,
                '4' => Rank.Four,
                '5' => Rank.Five,
                '6' => Rank.Six,
                '7' => Rank.Seven,
                '8' => Rank.Eight,
                '9' => Rank.Nine,
                '1' => Rank.Ten,
                'J' => Rank.Jack,
                'Q' => Rank.Queen,
                'K' => Rank.King,
                'A' => Rank.Ace,
            };

        private static Suit ParseSuit(string card) =>
            card[^1] switch
            {
                'H' => Suit.Hearts,
                'S' => Suit.Spades,
                'D' => Suit.Diamonds,
                'C' => Suit.Clubs,
            };
    }
}
```

This approach implements the [`IComparer<T>` interface][icomparer] to compare hands.
For more information, check the [`IComparer<T>` approach][approach-icomparer].

## Which approach to use?

What approach to choose depends a bit on how one would want the code to be used elsewhere.

[approach-integer-score]: https://exercism.org/tracks/csharp/exercises/poker/approaches/integer-score
[approach-icomparer]: https://exercism.org/tracks/csharp/exercises/poker/approaches/icomparer
[icomparer]: https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.icomparer-1
