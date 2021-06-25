# hint [![Hackage](https://img.shields.io/hackage/v/hint.svg)](https://hackage.haskell.org/package/hint) [![Build Status](https://github.com/haskell-hint/hint/workflows/CI/badge.svg)](https://github.com/haskell-hint/hint/actions)

This library defines an Interpreter monad within which you can interpret
strings like `"[1,2] ++ [3]"` into values like `[1,2,3]`. You can easily
exchange data between your compiled program and your interpreted program, as
long as the data has a `Typeable` instance.

You can choose which modules should be in scope while evaluating these
expressions, you can browse the contents of those modules, and you can ask for
the type of the identifiers you're browsing.

It is, essentially, a huge subset of the GHC API wrapped in a simpler API.

## Limitations

It is possible to run the interpreter inside a thread, but on GHC 8.8 and
below, you can't run two instances of the interpreter simultaneously.

GHC must be installed on the system on which the compiled executable is running.

## Example
```haskell
{-# LANGUAGE LambdaCase, ScopedTypeVariables, TypeApplications #-}
import Control.Exception (throwIO)
import Control.Monad (when)
import Control.Monad.Trans.Class (lift)
import Control.Monad.Trans.Writer (execWriterT, tell)
import Data.Foldable (for_)
import Data.List (isPrefixOf)
import Data.Typeable (Typeable)
import qualified Language.Haskell.Interpreter as Hint

-- |
-- Interpret expressions into values:
--
-- >>> eval @[Int] "[1,2] ++ [3]"
-- Right [1,2,3]
-- 
-- Send values from your compiled program to your interpreted program by
-- interpreting a function:
--
-- >>> Right f <- eval @(Int -> [Int]) "\\x -> [1..x]"
-- >>> f 5
-- [1,2,3,4,5]
eval :: forall t. Typeable t
     => String -> IO (Either Hint.InterpreterError t)
eval s = Hint.runInterpreter $ do
  Hint.setImports ["Prelude"]
  Hint.interpret s (Hint.as :: t)

-- |
-- >>> :{
-- do Right contents <- browse "Prelude"
--    for_ contents $ \(identifier, tp) -> do
--      when ("put" `isPrefixOf` identifier) $ do
--        putStrLn $ identifier ++ " :: " ++ tp
-- :}
-- putChar :: Char -> IO ()
-- putStr :: String -> IO ()
-- putStrLn :: String -> IO ()
browse :: Hint.ModuleName -> IO (Either Hint.InterpreterError [(String, String)])
browse moduleName = Hint.runInterpreter $ do
  Hint.setImports ["Prelude", "Data.Typeable", moduleName]
  exports <- Hint.getModuleExports moduleName
  execWriterT $ do
    for_ exports $ \case
      Hint.Fun identifier -> do
        tp <- lift $ Hint.typeOf identifier
        tell [(identifier, tp)]
      _ -> pure ()  -- skip datatypes and typeclasses
```

Check [example.hs](examples/example.hs) for a longer example (it must be run
from hint's base directory).
