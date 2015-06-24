# Writing a parser for a type isomorphic to Nat

<intro>
motivation, goal
<end intro>

The code is available on [github](https://github.com/Barbichu/ssrHoTT/).

Let us define a new type for natural integers in Type. We work in the subdirectory *theories*, as is often the case.

<!-- Coq code -->
    Inductive Nat : Type := O : Nat | S : Nat -> Nat.
<!-- End Coq code -->

We would like to be able to parse integers directly from a Coq script as Nats, for example

    Check 3%Nat.

should build the term

    S(S(S(O))) : Nat.

For this, we need a *plugin*, which we will call *ssrhott_nat_syntax* and which we will write in OCaml.

<!-- insert mention of merlin and .merlin syntax to be able to see the type of expressions
https://github.com/the-lambda-church/merlin  -->

We start by opening some Ocaml modules from the Coq sources, namely:

	open Glob_term
	open Bigint
	open Pp
	open Errors

	open Names
	open Globnames

	open Term

<explain more precisely what those modules are?>

For what follows, we need the handy function:

	let get_const dir s =
		Coqlib.find_reference "SsrHoTT.SsrHoTT_nat_syntax" dir s

which, from a path *dir* and a string *s* will build an object of type

	Globnames.global_reference
<!-- which is described in the kernel as "Global reference is a kernel side type for all references together". -->

which is a global representation of objects such as constants, constructors, variables or inductive types. <!-- cf library/globnames -->
We will need this representation to build the terms we parse, and indeed:

	let glob_O = get_const ["SsrHoTT";"nat"] "O"
	let glob_S = get_const ["SsrHoTT";"nat"] "S"

rebuild the constants for zero and successor.


We also have to build the type Nat as a global reference: <!-- some details needed here, probably after the code is cleaned up -->

	let nat_kn = make_path nat_definitions "Nat"
	let glob_Nat = ConstRef nat_kn

Now we want to write a function *nat_of_int* which will build terms as it goes from the Ocaml int's that it parses:

	let nat_of_int dloc n =
      if is_pos_or_zero n then begin
	    let ref_O = GRef (dloc, glob_O, None) in
	    let ref_S = GRef (dloc, glob_S, None) in
	    let rec mk_nat acc n =
          if n <> zero then
            mk_nat (GApp (dloc,ref_S, [acc])) (sub_1 n)
          else
            acc
	    in
	      mk_nat ref_O n
        end
      else
	    user_err_loc (dloc, "nat_of_int",
		str "Cannot interpret a negative number as a number of type nat")
		
The code is quite self-explanatory once we get past some heavy notations: we first test whether n is nonnegative (in the opposite case we throw an error as we are trying to parse negative numbers). Then we build a recursive function which will add the "S" constructor as long as its argument is greater than 0, and return its value otherwise.


We also need a printing function so that when we print the result of some computation involving numbers of type Nat, we can get it back as a usual numeral, thus

	Check (S O).

will answer

	1%Nat : Nat

This printing function is defined by

	exception Non_closed_number

	let rec int_of_nat = function
      | GApp (_,GRef (_,s,_),[a]) when Globnames.eq_gr s glob_S ->
    add_1 (int_of_nat a)
      | GRef (_,z,_) when Globnames.eq_gr z glob_O -> zero
      | _ -> raise Non_closed_number

	let uninterp_nat p =
      try
        Some (int_of_nat p)
	  with
        Non_closed_number -> None

Now the last step is to declare our new combination of parser and printer to Coq:

	let _ =
	  Notation.declare_numeral_interpreter "Nat_scope"
       (nat_path,["SsrHoTT";"nat"])
        nat_of_int
       ([GRef (Loc.ghost,glob_S,None);
         GRef (Loc.ghost,glob_O,None)], uninterp_nat, true)


(for later)
<!-- Coq code -->
Declare ML Module "ssrhott_nat_syntax_plugin".
<!-- End Coq code -->

